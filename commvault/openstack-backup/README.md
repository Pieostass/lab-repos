# Backup & Restore máy ảo OpenStack bằng Commvault

> **Stack:** OpenStack Epoxy | Ubuntu 24.04 LTS | Commvault 11.40  
> Tài liệu chi tiết kèm sơ đồ hạ tầng, IP thực tế và bảng tra cứu lỗi — dành cho người mới bắt đầu.

---

## 1. Sơ đồ hạ tầng

```
CommServe (Windows, 10.0.90.240)
        |
        |  điều phối job / API
        v
VSA Proxy / Access Node (Linux, cv-vsa-proxy, 10.0.90.160)
        |
        |  gọi API: Keystone:5000, Nova:8774, Glance:9292, Neutron:9696
        v
OpenStack Controller (10.0.90.241)
        |
        +-- cirros.tiny-1  (10.0.90.210)
        +-- cirros.tiny-2  (10.0.90.198)
```

### Bảng địa chỉ IP

| Thành phần | Loại máy | IP | Ghi chú |
|-----------|----------|-----|---------|
| CommServe | Windows Server | 10.0.90.240 | Commvault All-in-One (tên: pieost) |
| OpenStack Controller | Linux | 10.0.90.241 | Keystone, Nova, Glance, Neutron, Placement |
| VSA Proxy | Linux Ubuntu (cv-vsa-proxy) | 10.0.90.160 | Access Node — gọi API OpenStack, thực hiện Hot-Add backup |
| VM backup #1 | Cirros (cirros.tiny-1) | 10.0.90.210 | Trạng thái ACTIVE |
| VM backup #2 | Cirros (cirros.tiny-2) | 10.0.90.198 | Trạng thái ACTIVE |

---

## 2. Chuẩn bị môi trường

### 2.1 Trên VSA Proxy (10.0.90.160)

```bash
# Cài gói phụ trợ
sudo apt update
sudo apt install -y qemu-utils open-iscsi python3-openstackclient cryptsetup

# Kiểm tra các gói đã có
dpkg -l | grep -E "python3-openstackclient|qemu-utils|open-iscsi|cryptsetup"
which openstack

# Đồng bộ thời gian
sudo timedatectl set-ntp true
sudo systemctl restart chrony
timedatectl status   # kiểm tra "System clock synchronized: yes"

# Kiểm tra kết nối tới Controller
nc -zv 10.0.90.241 5000    # Keystone → phải trả về "succeeded"
nc -zv 10.0.90.241 8774    # Nova
nc -zv 10.0.90.241 9292    # Glance
nc -zv 10.0.90.241 9696    # Neutron
nc -zv 10.0.90.241 8778    # Placement
```

Cài Commvault VSA Agent:
```bash
sudo ./cvpkgadd
# CommServe Host Name: 10.0.90.240
# Client Name: cv-vsa-proxy
# Gói cài: Virtual Server Agent (VSA) + MediaAgent

# Kiểm tra sau khi cài
sudo commvault status
```

### 2.2 Trên CommServe Windows (10.0.90.240)

```powershell
# Kiểm tra dịch vụ Commvault
Get-Service | Where-Object { $_.Name -like "GxCVD*" -or $_.Name -like "Gx*" }

# Kiểm tra kết nối
Test-NetConnection -ComputerName 10.0.90.160 -Port 8403
Test-NetConnection -ComputerName 10.0.90.241 -Port 5000

# Đồng bộ thời gian
w32tm /resync
w32tm /query /status
```

### 2.3 Trên OpenStack Controller (10.0.90.241)

```bash
# Source file môi trường
find / -name "*openrc*" 2>/dev/null   # tìm file admin-openrc
source /home/hieu/admin-openrc

# Kiểm tra VM đang chạy
openstack server list --all-projects
# Kết quả mong đợi: thấy cv-vsa-proxy, cirros.tiny-1, cirros.tiny-2 — trạng thái ACTIVE

# ⚠️ QUAN TRỌNG: Kiểm tra Service Catalog trả về IP hay tên miền
openstack catalog list
# Nếu endpoint dạng "http://controller:8774" → phải cấu hình /etc/hosts (xem Mục 5)
```

### Checklist trước khi tiếp tục

- [ ] Proxy đã cài Commvault VSA + MediaAgent, `commvault status` đang chạy
- [ ] Proxy đã cài đủ: `qemu-utils`, `open-iscsi`, `python3-openstackclient`, `cryptsetup`
- [ ] Proxy gọi thông các port 5000/8774/9292/9696/8778 sang Controller (`nc -zv`)
- [ ] CommServe — tất cả dịch vụ `Gx*` đang Running
- [ ] NTP đã đồng bộ trên cả 3 máy
- [ ] `openstack server list --all-projects` ra đủ danh sách VM
- [ ] Đã xác định đúng **Project Name** chứa VM (ví dụ: `admin`)
- [ ] Đã kiểm tra `openstack catalog list` — endpoint dạng IP hay tên miền

---

## 3. Cấu hình Commvault

### 3.1 Thêm Hypervisor OpenStack

Vào **Protect → Virtualization → Hypervisors → Add Hypervisor → OpenStack**:

| Field | Giá trị |
|-------|---------|
| Auth URL | `http://10.0.90.241:5000/v3` (hoặc `http://controller:5000/v3` sau khi fix hosts) |
| Username | `admin` |
| Project Name | `admin` *(phải đúng project chứa VM)* |
| Domain name | `Default` |
| Access Node | `cv-vsa-proxy` (10.0.90.160) |
| Tên Hypervisor | `Commcell_Backup-OpenStack-Hieu` |

> ⚠️ **Project Name sai** → Commvault xác thực Token thành công (HTTP 201) nhưng danh sách VM **hoàn toàn trống** → lỗi [91:479] hoặc [91:587].

### 3.2 Tạo VM Group

Vào **Protect → Virtualization → VM Groups → Add VM Group**:

- Chọn Hypervisor vừa tạo
- Content → Add → **Instances** (ép gọi API tươi sang OpenStack)
- Tick chọn: `cirros.tiny-1` và `cirros.tiny-2`
- Access nodes → Edit → chọn **cv-vsa-proxy** (10.0.90.160)

---

## 4. Thực hiện Backup

1. Vào **Protect → Virtualization → VM Groups** → chọn nhóm
2. Bấm **Backup Now** → chọn **Full**
3. Submit → theo dõi tại **Monitoring → Jobs**
4. Status `Completed` (màu xanh) = backup thành công ✅

---

## 5. Thực hiện Restore

### 5.1 Restore Full VM (In-place)

1. **Monitoring → Jobs** → tìm job backup → dấu ba chấm → **Restore**
2. Tick VM cần khôi phục → chọn **In place** + **Overwrite if VM already exists**
3. Access Node: `cv-vsa-proxy` (10.0.90.160)
4. Submit → kiểm tra trên Horizon sau khi Completed

### 5.2 Guest File Restore (chỉ lấy file lẻ)

- Chọn kiểu **Guest Files** thay vì Full VM
- Commvault mount đĩa ảo từ backup, hiển thị cây thư mục Linux
- Duyệt → tick file cần lấy → Restore (Download hoặc khôi phục vào VM)

### 5.3 Cross-Platform Restore (OpenStack → VMware ESXi)

Giao diện Web rút gọn thường khoá tính năng này. Dùng **CommCell Console** (Java):

```
Client Computers → Commcell_Backup-OpenStack-Hieu
  → Virtual Server → OpenStack → defaultSubclient
  → Chuột phải → Browse and Restore → View Content
  → Recover All Selected → Destination Client: vCenter/ESXi
```

> **Lưu ý:** Cirros thiếu driver VMXNET3 của VMware. Nếu mất mạng sau restore, đổi Network Adapter sang **E1000** trong cấu hình phần cứng ESXi.

---

## 6. Lỗi thường gặp & cách xử lý

### ⚡ Lỗi quan trọng nhất: Token OK nhưng danh sách VM trống [91:587]

**Nguyên nhân:** Keystone trả về endpoint dạng tên miền trong Service Catalog:
```
"url": "http://controller:8774/v2.1"   # Nova
"url": "http://controller:9292"        # Glance
```
Proxy và CommServe không phân giải được tên `controller` → gọi API thất bại âm thầm.

**Kiểm tra:**
```bash
# Chạy trên Proxy (10.0.90.160)
curl -s -X POST http://10.0.90.241:5000/v3/auth/tokens \
  -H "Content-Type: application/json" \
  -d '{"auth":{"identity":{"methods":["password"],"password":{"user":{"name":"admin","domain":{"name":"Default"},"password":"admin123"}}},"scope":{"project":{"name":"admin","domain":{"name":"Default"}}}}}' \
  | python3 -m json.tool | grep '"url"'
# Nếu thấy "http://controller:..." → cần fix hosts
```

**Fix:**
```bash
# Trên Proxy Linux (10.0.90.160)
echo "10.0.90.241 controller" | sudo tee -a /etc/hosts
ping -c 3 controller   # kiểm tra

# Trên CommServe Windows (10.0.90.240) — CMD với quyền Admin
echo 10.0.90.241 controller >> C:\Windows\System32\drivers\etc\hosts
type C:\Windows\System32\drivers\etc\hosts   # kiểm tra

# Restart Commvault trên Proxy
sudo commvault restart
```

---

### Bảng tra cứu lỗi đầy đủ

| Lỗi / Thông báo | Nguyên nhân | Cách fix |
|----------------|-------------|----------|
| `[91:479]` There are no instances in the subclient content | VM Group trống do: sai Project Name, cache chưa refresh, hoặc link định danh VM bị đứt | VM Group → Content → Manage → Remove VM cũ → Add → Instances (chọn lại) |
| `[91:587]` No instances were discovered | UUID VM trong Commvault không khớp OpenStack hiện tại, hoặc lỗi DNS endpoint | Tạo VM Group mới để ép quét lại từ đầu; kiểm tra `/etc/hosts` |
| Token OK nhưng VM list trống | Endpoint dạng tên miền, Proxy chưa resolve được | Fix `/etc/hosts` trên Proxy + CommServe (xem Mục 6 trên) |
| `Error 0x908: Invalid XML Input` | Cấu trúc XML CLI không khớp schema Commvault hiện tại | Dùng giao diện Web thay CLI |
| `you must be uid=0` khi restart Commvault | Thiếu `sudo` | `sudo commvault restart` |
| `source admin-openrc: No such file` | Đứng sai thư mục | `find / -name "*openrc*" 2>/dev/null` → source đúng đường dẫn |
| `Add-Content is not recognized` trên Windows | Đang ở CMD không phải PowerShell | Dùng CMD: `echo 10.0.90.241 controller >> C:\Windows\System32\drivers\etc\hosts` |
| Snapshot kẹt `error_deleting` | Mạng chập chờn, volume quá lớn | `openstack volume snapshot set --status error <UUID>` rồi delete |
| Job treo 0% (Hot-Add thất bại) | Proxy thiếu `open-iscsi` hoặc không thông Storage Network | `sudo apt install open-iscsi`; kiểm tra `/etc/iscsi/initiatorname.iscsi` |
| `403 Forbidden` khi gọi API snapshot | Tài khoản API thiếu quyền trong policy.yaml | Dùng tài khoản `admin` hoặc chỉnh `policy.yaml` trên Controller |
| Token expired ngay sau khi cấp | Lệch giờ NTP giữa các máy | `sudo timedatectl set-ntp true && sudo systemctl restart chrony` trên tất cả máy |
| VM restore xong mất mạng | Network ID trong bản backup không còn tồn tại | Chọn Advanced Restore → map thủ công sang mạng mới |
| VM Cirros restore sang VMware mất mạng | Thiếu driver VMXNET3 | Đổi Network Adapter sang E1000 trong VMware |
| Ô "Restore as" không hiện VMware | Cross-Platform bị khoá ở giao diện Web cơ bản | Dùng CommCell Console (Java) để restore |

---

## 7. Kịch bản nâng cao (tham khảo)

- **Multi-Tenant Backup:** phân quyền backup/restore riêng theo từng Project
- **Live Sync / DR Replication:** đồng bộ VM sang cụm OpenStack dự phòng tự động
- **Backup Cinder Volume độc lập:** chỉ backup ổ dữ liệu, không cần backup OS
- **Migration xuyên nền tảng:** dùng Commvault chuyển VM giữa VMware ↔ OpenStack

---

*Tài liệu được tổng hợp từ thực nghiệm triển khai thực tế — OpenStack Epoxy + Commvault 11.40 trên Ubuntu 24.04 LTS*
