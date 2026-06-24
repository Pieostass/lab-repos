# 📚 Lab Repositories - Tài Liệu Triển Khai & Thực Chiến Hệ Thống

Chào mừng đến với kho lưu trữ các bài lab thực chiến về quản trị hệ thống, mạng, backup & restore, và giám sát hạ tầng (Monitoring). Kho lưu trữ này được tổng hợp từ các cấu hình và triển khai thực tế.

---

## 🛠 Danh Sách Các Bài Lab

Kho lưu trữ bao gồm các chuyên đề chính sau:

### 1. 💾 [Commvault Backup Labs](./commvault/)
Tài liệu thực chiến về triển khai backup & restore với Commvault v11.40 cho các hệ thống:
* **OpenStack VM Backup:** Thực hiện backup máy ảo trên OpenStack Epoxy.
* **Kubernetes Backup:** Giải pháp sao lưu cụm K8s v1.31.
* **Oracle Database Backup:** Sao lưu cơ sở dữ liệu Oracle XE 21c.
* Chi tiết cấu hình môi trường, IP và các bước thiết lập.

### 2. 🛡️ [FortiGate Network Configuration](./fortigate-config/)
Cấu hình chi tiết thiết bị tường lửa FortiGate 100D (FortiOS 6.0.18) cho môi trường văn phòng đa tầng:
* Thiết lập định tuyến (Routing), Inter-VLAN.
* Cấu hình NAT (Source NAT, Destination NAT / Virtual IP).
* Xây dựng chính sách bảo mật (Firewall Policies) và phân quyền truy cập.

### 3. ☁️ [MinIO & Commvault S3 Integration](./minio-commvault/)
Giải pháp xây dựng cụm lưu trữ MinIO Distributed Object Storage trên Ubuntu 24.04 LTS tích hợp với Commvault làm Cloud Backup Target (S3 Compatible):
* Triển khai cụm MinIO 2 nodes, 8 drives với Erasure Coding EC:4.
* Cấu hình Commvault Cloud Storage Library kết nối đến MinIO S3 API.
* Khắc phục các lỗi chứng chỉ SSL/TLS tự ký (Self-signed Certificate).

### 4. 📊 [Observium Network Monitoring](./observium-monitoring/)
Hướng dẫn cài đặt và cấu hình hệ thống giám sát thiết bị mạng Observium Community Edition trên nền tảng Ubuntu 24.04 LTS:
* Giám sát qua giao thức SNMP v2c/v3 cho Switch AirPro và máy chủ Linux.
* Khắc phục lỗi tiến trình apt/dpkg lock và lỗi `/dev/fd` trên môi trường ảo hóa VMware.

### 5. 📈 [Zabbix Monitoring Server & Agent](./zabbix-monitoring/)
Triển khai hệ thống giám sát Zabbix 7.0 LTS và Zabbix Agent trên Ubuntu 24.04 LTS:
* Cài đặt thủ công Zabbix Server với MySQL 8.0, Apache2 và PHP.
* Cài đặt tự động Zabbix Agent trên các máy chủ được giám sát bằng Ansible.
* Cấu hình giám sát tài nguyên (CPU, RAM, Disk, Network) và thiết lập cảnh báo (Triggers).

---

## ⚙️ Hướng Dẫn Sử Dụng
Mỗi thư mục con đều có một file `README.md` riêng, cung cấp hướng dẫn chi tiết từng bước, sơ đồ kiến trúc hệ thống (Topology), và các đoạn mã/cấu hình mẫu tương ứng. Bạn chỉ cần truy cập vào từng thư mục để bắt đầu tham khảo và triển khai.
