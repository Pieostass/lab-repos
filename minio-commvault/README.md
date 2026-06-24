# MinIO + Commvault Integration Lab

Triển khai MinIO Distributed Object Storage tích hợp với Commvault làm Cloud Backup Target (S3 Compatible).

## Lab Environment

| Node | IP | OS |
|------|----|----|
| node1 (MinIO) | 10.0.90.214 | Ubuntu 24.04 LTS |
| node2 (MinIO) | 10.0.90.215 | Ubuntu 24.04 LTS |
| Commvault CommServe | 10.0.90.210 | Windows Server 2019 |

**Storage:** 4 ổ × 10 GB loop device/node → tổng 80 GB, Erasure Coding EC:4

## Architecture

```
Commvault CommServe
       │
       │  S3 Compatible API (port 9000)
       ▼
┌─────────────┐     ┌─────────────┐
│   node1     │◄───►│   node2     │
│ 10.0.90.214 │     │ 10.0.90.215 │
│ 4 drives    │     │ 4 drives    │
└─────────────┘     └─────────────┘
   MinIO Distributed Mode (EC:4)
```

## Steps Thực Hiện

### 1. Chuẩn bị Storage (Loop Device)
```bash
sudo mkdir -p /mnt/drive-{1,2,3,4}
sudo mkdir -p /opt/minio-disks

for i in 1 2 3 4; do
  sudo dd if=/dev/zero of=/opt/minio-disks/disk${i}.img bs=1M count=10240
done

for i in 1 2 3 4; do
  LOOP=$(sudo losetup --find --show /opt/minio-disks/disk${i}.img)
  sudo mkfs.xfs -f $LOOP
  sudo mount $LOOP /mnt/drive-${i}
done
```

### 2. Cài đặt MinIO
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && sudo mv minio /usr/local/bin/

wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/mc
```

### 3. Cấu hình `/etc/default/minio` (cả 2 node)
```bash
MINIO_VOLUMES="http://node{1...2}/mnt/drive-{1...4}"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_SITE_NAME=my-minio
```

### 4. Tạo Access Key cho Commvault
```bash
mc alias set myminio http://10.0.90.214:9000 minioadmin minioadmin123
mc admin accesskey create myminio
mc mb myminio/commvault-backup
```

### 5. Tích hợp vào Commvault
- Command Center → **Storage → Cloud → Add cloud storage**
- Type: **S3 Compatible Storage**
- Service Host: `10.0.90.214:9000`
- Điền Access Key / Secret Key từ bước trên

## Kết Quả Kiểm Tra

| Test Case | Kết quả |
|-----------|---------|
| Tắt 1 node, đọc dữ liệu từ node còn lại | ✅ Pass |
| Ghi dữ liệu khi 1 node down | ✅ Pass |
| Khởi động lại node, tự heal | ✅ Pass |
| Umount 1 ổ đĩa (disk failure simulation) | ✅ Pass (EC:4 chịu được mất 4 ổ) |

## Key Learnings

- MinIO Distributed Mode yêu cầu số ổ đĩa chẵn, tối thiểu 4 ổ
- Erasure Coding EC:4 trên 8 ổ (2 node × 4): chịu được mất tối đa 4 ổ mà không mất dữ liệu
- Commvault kết nối MinIO qua S3 Compatible API — không cần AWS, tiết kiệm chi phí
- MinIO Community Edition không có UI quản lý Access Key → dùng `mc admin accesskey create`

## Tech Stack

`MinIO` `Commvault` `Ubuntu 24.04` `VMware` `S3 Compatible Storage` `Object Storage`
