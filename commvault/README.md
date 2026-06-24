#  Commvault Backup Labs

Tài liệu thực chiến về triển khai backup & restore với Commvault — xây dựng từ kinh nghiệm lab thực tế.

##  Labs

| Lab | Stack | Tài liệu |
|-----|-------|----------|
| OpenStack VM Backup | OpenStack Epoxy + Commvault 11.40 | [Xem →](./openstack-backup/) |
| Kubernetes Backup | K8s v1.31 + Commvault 11.40 | [Xem →](./kubernetes-backup/) |
| Oracle Database Backup | Oracle XE 21c + Commvault 11.40 | [Xem →](./oracle-backup/) |

##  Môi trường chung

| Thành phần | IP | Vai trò |
|------------|-----|---------|
| Commvault CommServe | 10.0.90.240 | Quản lý toàn bộ job backup |
| OpenStack Controller | 10.0.90.241 | Controller node |
| VM K8s / Oracle | 10.0.90.244 / 10.0.90.243 | Workload servers |
| VSA Proxy | 10.0.90.160 | Access node cho OpenStack |

##  Stack kỹ thuật

- **Virtualization:** VMware vSphere/ESXi, OpenStack Epoxy
- **Backup:** Commvault 11.40
- **Platforms:** Ubuntu 24.04 LTS, Windows Server
- **Databases:** Oracle XE 21c, MySQL, MariaDB
- **Container:** Kubernetes v1.31.14 (kubeadm)

##  Điểm nổi bật

Mỗi lab đều có:
- Sơ đồ kiến trúc + bảng IP thực tế
- Hướng dẫn từng bước với kết quả mong đợi
- **Phần xử lý lỗi thực tế** — lỗi gặp trực tiếp khi làm lab, không phải lý thuyết
- Checklist trước khi chạy backup

---
*Tất cả tài liệu được tổng hợp từ thực nghiệm triển khai thực tế.*
