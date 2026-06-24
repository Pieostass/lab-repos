# Observium Network Monitoring Lab

Triển khai Observium Community Edition trên Ubuntu 24.04 để giám sát thiết bị mạng qua giao thức SNMP.

## Environment

| Thành phần | Thông tin |
|------------|-----------|
| Observium Server | Ubuntu 24.04 LTS – 192.168.136.154 |
| Phiên bản | Observium CE 26.1.14545 |
| Ngày thực hiện | 15/05/2026 |
| Thiết bị giám sát | Switch AirPro, Linux hosts |

## Cài Đặt Observium

### 1. Chạy script cài đặt
> **Lưu ý:** Môi trường VMware gặp lỗi `/dev/fd` khi dùng process substitution. Phải tải script về rồi chạy thay vì pipe trực tiếp.

```bash
wget https://www.observium.org/observium_installscript.sh
sudo bash ./observium_installscript.sh
# Chọn option 1: Community Edition
```

### 2. Xử lý lỗi dpkg lock (VMware environment)
```bash
# Nếu gặp: E: Could not get lock /var/lib/dpkg/lock-frontend
sudo lsof /var/lib/dpkg/lock-frontend  # tìm PID
sudo kill -9 <PID>
sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
sudo dpkg --configure -a
sudo apt install snmpd -y
```

### 3. Add localhost thủ công (sau khi fix lỗi snmpd)
```bash
cd /opt/observium
sudo ./add_device.php localhost public v2c
sudo ./discovery.php -h all
sudo ./poller.php -h all
```

**Web UI:** http://192.168.136.154 | admin / (mật khẩu tạo lúc cài)

## Cấu Hình SNMP

### Trên mỗi host/thiết bị cần giám sát:
```bash
sudo apt install snmpd -y
sudo nano /etc/snmp/snmpd.conf
```

Chỉnh 2 dòng sau:
```
# Cho phép lắng nghe tất cả interface (không chỉ localhost)
agentaddress udp:161,udp6:161

# Bỏ -V systemonly để Observium đọc được đầy đủ thông tin
rocommunity public default
```

```bash
sudo systemctl restart snmpd && sudo systemctl enable snmpd

# Kiểm tra đang lắng nghe đúng (phải thấy 0.0.0.0 thay vì 127.0.0.1)
sudo ss -ulnp | grep 161
```

### Thêm thiết bị vào Observium
```bash
cd /opt/observium
sudo ./add_device.php <IP_thiết_bị> public v2c
sudo ./discovery.php -h all
sudo ./poller.php -h all
```

## Kết Quả

- Giám sát được: CPU, RAM, disk I/O, network traffic, uptime trên tất cả Linux host
- Giám sát switch AirPro qua SNMP: port status, bandwidth, VLAN
- Biểu đồ lưu lượng real-time theo từng interface

## Troubleshooting

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `dpkg lock` khi cài | `unattended-upgrades` đang chạy | Kill process, xóa lock file |
| SNMP chỉ trả về system info | `-V systemonly` trong config | Bỏ tham số này trong `rocommunity` |
| Device không hiện metric | snmpd chỉ bind localhost | Đổi `agentaddress` sang `udp:161` |

## Key Learnings

- Observium CE miễn phí, không cần tài khoản, phù hợp lab và SMB
- SNMP v2c dùng community string `public` — nên đổi trong môi trường production
- `-V systemonly` là cấu hình mặc định Ubuntu, phải bỏ để monitoring tool hoạt động đầy đủ
- Poller chạy qua cron mỗi 5 phút — có thể chạy thủ công để test ngay

## Tech Stack

`Observium CE` `Ubuntu 24.04` `SNMP v2c` `PHP` `MySQL` `AirPro Switch`
