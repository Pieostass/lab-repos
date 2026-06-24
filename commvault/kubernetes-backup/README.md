# Backup & Restore Kubernetes bằng Commvault

> **Stack:** Ubuntu 24.04 LTS | Kubernetes v1.31.14 (kubeadm) | Commvault 11.40  
> Hướng dẫn từ đầu: dựng K8s cluster, tích hợp Commvault, backup & restore namespace + PVC.

---

## 1. Kiến trúc Lab

```
CommServe (10.0.90.240)
    |
    | điều phối job
    v
VM K8s (10.0.90.244) — hieu-k8s-linux
    ├── Kubernetes v1.31.14 (single-node, kubeadm)
    ├── Commvault iDataAgent (Access Node)
    └── Namespace: test-backup
            ├── Deployment: test-app (nginx)
            └── PVC: test-pvc (2Gi, local-path)
```

### Thông số môi trường

| Thành phần | Phiên bản | Ghi chú |
|-----------|----------|---------|
| Ubuntu | 24.04.4 LTS (Noble) | OS của VM K8s |
| Kubernetes | v1.31.14 (kubeadm) | Single-node cluster |
| Container Runtime | containerd.io (Docker repo) | SystemdCgroup = true |
| CNI | Flannel | Pod network 10.244.0.0/16 |
| Storage | local-path-provisioner | PVC storage class |
| Commvault | 11.40 | iDataAgent + K8s backup |
| RAM | 16 GB | Tối thiểu cho K8s lab |
| CPU | 8 vCPU | |
| Disk | 100 GB | |

---

## 2. Cài đặt Kubernetes trên Ubuntu 24.04

### Bước 1: Kiểm tra VM

```bash
cat /etc/os-release | grep -E 'NAME|VERSION'
free -h && nproc
df -h
ip addr show | grep 'inet '
swapon --show
```

### Bước 2: Tắt Swap

> ⚠️ Kubernetes yêu cầu tắt swap hoàn toàn — nếu còn swap, kubelet sẽ không start được.

```bash
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
swapon --show   # không có output = thành công
```

### Bước 3: Load Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

lsmod | grep -E 'overlay|br_netfilter'
```

### Bước 4: Set Kernel Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
sysctl net.ipv4.ip_forward   # kết quả mong đợi: net.ipv4.ip_forward = 1
```

### Bước 5: Cài containerd (từ Docker repo)

> ⚠️ **Không** cài containerd từ apt Ubuntu — version cũ, thiếu SystemdCgroup. Phải dùng Docker repo.

```bash
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu noble stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y containerd.io
```

### Bước 6: Cấu hình containerd

> ⚠️ Bước `SystemdCgroup = true` **cực kỳ quan trọng** — bỏ qua sẽ kubelet crash loop vĩnh viễn.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

grep SystemdCgroup /etc/containerd/config.toml   # phải thấy: SystemdCgroup = true

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd | grep -E 'Active|enabled'
```

### Bước 7: Cài kubeadm, kubelet, kubectl

> ⚠️ Phải viết repo trên **1 dòng duy nhất** — xuống dòng sẽ báo lỗi "Malformed entry".

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version
kubectl version --client
```

### Bước 8: Init Cluster

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.0.90.244 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

Kết quả mong đợi: `Your Kubernetes control-plane has initialized successfully!`

### Bước 9: Setup kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Bước 10: Cài CNI Flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Bước 11: Bỏ taint control-plane (single node)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Bước 12: Cài Storage Class (local-path-provisioner)

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
# Kết quả: NAME                   PROVISIONER             RECLAIMPOLICY
#          local-path (default)   rancher.io/local-path   Delete
```

### Bước 13: Kiểm tra cluster

```bash
# Đợi ~60 giây
kubectl get nodes
kubectl get pods -A
# Kết quả mong đợi: node Ready, tất cả pod Running
```

---

## 3. Deploy App test với PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: test-backup
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: test-backup
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: test-backup
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - mountPath: /data
          name: data-vol
      volumes:
      - name: data-vol
        persistentVolumeClaim:
          claimName: test-pvc
EOF

# Đợi pod Running
kubectl get pods -n test-backup -w

# Ghi data test vào PVC
POD=$(kubectl get pod -n test-backup -o name | head -1)
kubectl exec -n test-backup $POD -- \
  sh -c "echo 'du lieu truoc khi backup' > /data/test.txt && cat /data/test.txt"
```

---

## 4. Cấu hình Commvault backup Kubernetes

### 4.1 Push Commvault Agent lên VM K8s

Từ Command Center (http://10.0.90.240/commandcenter):

**Manage → Infrastructure → Servers → Add server**

| Field | Giá trị |
|-------|---------|
| Host name / IP | 10.0.90.244 |
| OS Type | Unix |
| Username | hieu |
| SSH port | 22 |
| Packages | File System |

### 4.2 Tạo Service Account cho Commvault

```bash
kubectl create serviceaccount k8s-backup -n kube-system

kubectl create clusterrolebinding k8s-backup-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:k8s-backup

# Tạo token có hạn 1 năm — copy toàn bộ token này
kubectl create token k8s-backup -n kube-system --duration=8760h
```

### 4.3 Configure Kubernetes trong Commvault

**Protect → Kubernetes → Configure Kubernetes** (5 bước):

**Bước 1 — Select Access Node:** chọn `hieu-k8s-linux` (10.0.90.244)

> ⚠️ Phải chọn đúng `hieu-k8s-linux` — nếu chọn `pieost` (CS) thì backup chỉ lưu manifest, **không backup được PVC data**.

**Bước 2 — Plan:** chọn plan backup có sẵn

**Bước 3 — Add Cluster:**

| Field | Giá trị |
|-------|---------|
| Kubernetes API server | `https://10.0.90.244:6443` |
| Name | `hieu-k8s` |
| Authentication type | Service account |
| Service account | `k8s-backup` |
| Service token | Paste token từ lệnh trên |
| etcd protection | Off (cho lab) |

**Bước 4 — Add Application Group:**

| Field | Giá trị |
|-------|---------|
| Application group name | `test-backup-group` |
| Content | Expand `hieu-k8s` → tick namespace `test-backup` |

---

## 5. Thực hiện Backup

```bash
# Kiểm tra trước khi backup
kubectl get pods -n test-backup
POD=$(kubectl get pod -n test-backup -o name | head -1)
kubectl exec -n test-backup $POD -- cat /data/test.txt
```

Trên Commvault:
1. **Protect → Kubernetes → hieu-k8s**
2. Click `...` → **Back up now**
3. Tick `test-backup-group` → NEXT → Submit
4. Theo dõi tại **Jobs**

**Kết quả backup thành công:**

| Thông số | Giá trị | Ý nghĩa |
|---------|---------|---------|
| Status | Completed | Backup hoàn thành |
| Access node | hieu-k8s-linux | Đúng access node |
| Backup size | ~2 GB | Có data thật, không phải chỉ manifest |
| Applications | 2 (Namespace + Deployment) | Cả namespace và workload |

> ⚠️ Nếu thấy `Data transferred = 0B` và Backup size = vài KB → sai access node. Đổi lại sang `hieu-k8s-linux` trong Configuration của cluster.

---

## 6. Thực hiện Restore

### Mô phỏng mất dữ liệu

```bash
kubectl delete namespace test-backup
kubectl get namespace test-backup   # phải báo NotFound
```

### Restore từ Commvault

1. **Protect → Kubernetes → hieu-k8s → Application groups**
2. Click `test-backup-group` → `...` → **Restore**
3. Chọn **Namespace and cluster level**
4. Tick namespace `test-backup` (Size: 2 GB) → RESTORE
5. Destination: **In place** | Access node: **hieu-k8s-linux**

**Thông số restore thực tế:**

| Thông số | Giá trị |
|---------|---------|
| Status | Completed |
| Duration | 1 phút 29 giây |
| No. of successes | 2 |
| Failures | 0 |

### Xác nhận restore thành công

```bash
kubectl get namespace test-backup
kubectl get pods -n test-backup
kubectl get pvc -n test-backup

# Kiểm tra data trong PVC còn không — quan trọng nhất
POD=$(kubectl get pod -n test-backup -o name | head -1)
kubectl exec -n test-backup $POD -- cat /data/test.txt
# Kết quả mong đợi: du lieu truoc khi backup ✅
```

---

## 7. Lỗi thường gặp & cách xử lý

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `Malformed entry` trong kubernetes.list | Dòng repo bị xuống dòng khi copy | Xóa file → viết lại trên 1 dòng duy nhất |
| `conntrack not found` | Package conntrack chưa cài trên Ubuntu 24.04 | `sudo apt install -y conntrack` |
| Pod Pending — `no storage class is set` | kubeadm không tự cài storage class | Cài `local-path-provisioner` (xem Bước 12) |
| Pod Pending — `untolerated taint control-plane` | Single-node chưa bỏ taint | `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` |
| Backup job overloaded `[19:2601]` | Access node là `pieost` (CS) không đủ tài nguyên | Cài agent lên VM K8s → đổi access node sang `hieu-k8s-linux` |
| Backup size = 3.45 KB, `Data transferred = 0B` | Access node sai — không mount được PVC | Đổi access node sang `hieu-k8s-linux` → chạy lại backup |
| `No applications were discovered [91:587]` | Đã xóa namespace trước khi backup | Tạo lại namespace + deploy lại app + ghi data trước khi backup |
| `MediaAgent not ready [32:690]` | MediaAgent được assign không sẵn sàng | Manage → Infrastructure → Media agents → kiểm tra status |

---

## 8. Checklist nhanh

### Trước khi backup

| # | Kiểm tra | Lệnh |
|---|---------|------|
| 1 | Namespace tồn tại | `kubectl get namespace test-backup` |
| 2 | Pod đang Running | `kubectl get pods -n test-backup` |
| 3 | PVC đang Bound | `kubectl get pvc -n test-backup` |
| 4 | Data trong PVC | `kubectl exec ... -- cat /data/test.txt` |
| 5 | Commvault agent online | Client có chấm xanh trong Command Center |
| 6 | Access node đúng | Phải là `hieu-k8s-linux`, không phải `pieost` |

### Thông tin quan trọng

| Thông tin | Giá trị |
|----------|---------|
| K8s cluster IP | 10.0.90.244 |
| K8s API server | https://10.0.90.244:6443 |
| Commvault CS IP | 10.0.90.240 |
| Namespace test | test-backup |
| Storage class | local-path |
| Commvault cluster name | hieu-k8s |
| Application group | test-backup-group |
| Access node | hieu-k8s-linux |
| Kubeconfig | `$HOME/.kube/config` |

---

*Tài liệu được tổng hợp từ thực nghiệm triển khai thực tế — Kubernetes v1.31.14 + Commvault 11.40 trên Ubuntu 24.04 LTS*
