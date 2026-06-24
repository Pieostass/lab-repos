# Commvault Deployment Guide – Dành Cho Kỹ Sư Mới

Hướng dẫn triển khai end-to-end Commvault Backup Solution trên VMware vSphere: từ tạo VM đến cấu hình backup database MySQL/MariaDB.

## Environment

| Thành phần | Thông tin |
|------------|-----------|
| Phiên bản Commvault | Platform Release 11.40 |
| Hypervisor | VMware vSphere / vCenter |
| CommServe OS | Windows Server 2019 |
| Database Server | Ubuntu 22.04 LTS |
| Deployment type | All-in-one (CommServe + MediaAgent) |

## Architecture

```
vCenter / ESXi Host
├── Resource Pool: TenBan-Resource-Pool
│   ├── CV-CommServe-Lab (Windows Server 2019)
│   │   ├── C: OS + Commvault (100GB)
│   │   ├── D: Commvault Index (100GB)
│   │   └── E: Backup Repository (500GB)
│   └── CV-MySQL-Lab (Ubuntu 22.04)
│       ├── MySQL 8.0
│       └── MariaDB
│
Commvault Command Center → https://IP-CommServe/commandcenter
```

## Bước 1: Tạo Resource Pool trên vSphere

```
vSphere Client → ESXi Host → chuột phải → New Resource Pool
- Tên: TenBan-Resource-Pool
- Memory Limit: 40,000 MB (tránh dùng hết RAM host)
- Reservation Type: Expandable
```

> **Quan trọng:** Tạo tất cả VM trong Resource Pool riêng để tránh xung đột tài nguyên với đồng nghiệp.

## Bước 2: Tạo VM CommServe

### Cấu hình phần cứng

| Thành phần | Giá trị | Ghi chú |
|------------|---------|---------|
| CPU | 4–8 vCPU | Tối thiểu 4 |
| RAM | 16 GB | Bắt buộc |
| Disk 1 | 100 GB Thin | OS + Commvault |
| Disk 2 | 100 GB Thin | Index/Data |
| Disk 3 | 500 GB Thin | Backup Repository |
| OS | Windows Server 2019 | Desktop Experience |

> **Lỗi thường gặp:** `ACPI motherboard layout requires EFI` → Edit Settings → VM Options → Boot Options → đổi Firmware từ **EFI sang BIOS**.

### Cài đặt Commvault (All-in-one)
```
Chạy Commvault_Media_11_40.exe với quyền Administrator
→ Select Roles: All in one
→ KHÔNG tick "Make this CommServe failover node"
→ Data Directory: D:\ (nếu có ổ D)
→ Chờ 15–30 phút
→ Reboot
→ Truy cập: https://IP-CommServe/commandcenter
```

### Cấu hình Backup Repository
```
Command Center → Storage → Disk → Local DiskStorage
→ Mount Paths → Add Mount Path → E:\
```

> **Lỗi:** `Mount path is below watermark` → Ổ E chưa được format trong Windows. Vào `diskmgmt.msc` format trước.

## Bước 3: Tạo VM Ubuntu (Database Server)

| Thành phần | Giá trị |
|------------|---------|
| Tên VM | CV-MySQL-Lab |
| CPU | 2–4 vCPU |
| RAM | 4 GB |
| Disk | 50 GB Thin |
| OS | Ubuntu Server 22.04 LTS |

```bash
# Sau khi cài Ubuntu
sudo apt update && sudo apt upgrade -y
sudo apt install open-vm-tools -y  # bắt buộc để hiện IP trên vSphere

# Cài MySQL
sudo apt install mysql-server -y

# Cài MariaDB (nếu cần test cả hai)
sudo apt install mariadb-server -y
```

## Bước 4: Kết Nối Commvault với vCenter

```
Command Center → Protect → Virtualization → Add hypervisor
→ Type: VMware vCenter
→ Host: IP vCenter
→ Credential: administrator@vsphere.local
→ Access node: CommServe (All-in-one)
```

## Bước 5: Cài Agent và Backup Database

```bash
# Trên Ubuntu VM, tải Commvault agent
# (copy từ CommServe hoặc tải qua download center)
sudo ./cvpkgadd

# Trong Command Center
→ Protect → Databases → Add server
→ Chọn MySQL/MariaDB
→ Test connection → Create backup plan
```

## Troubleshooting Thường Gặp

| Lỗi | Giải pháp |
|-----|-----------|
| ACPI EFI boot error | Đổi Firmware từ EFI sang BIOS trong VM settings |
| Mount path below watermark | Format ổ đĩa trong Windows trước |
| Agent không kết nối CommServe | Kiểm tra firewall, port 8400-8403 phải mở |
| vCenter discovery fail | Tắt Windows Firewall tạm thời để test |
| SQL Server Express install lâu | Bình thường, chờ 15-30 phút, không tắt |

## Key Learnings

- All-in-one deployment tiết kiệm tài nguyên trong môi trường lab
- Resource Pool quan trọng trong môi trường lab chia sẻ — tránh dùng hết RAM của ESXi host
- Commvault tự cài SQL Server Express trong quá trình cài đặt — không cần cài trước
- Disk E phải được format trong Windows trước khi Commvault nhận là mount path

## Tech Stack

`Commvault 11.40` `VMware vSphere` `Windows Server 2019` `Ubuntu 22.04` `MySQL 8.0` `MariaDB`
