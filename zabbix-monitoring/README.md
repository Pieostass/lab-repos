# Zabbix Monitoring Server & Agent Lab

Triển khai Zabbix 7.0 trên Ubuntu 24.04 để giám sát hạ tầng IT, bao gồm cài đặt thủ công và tự động bằng Ansible.

## Environment

| Thành phần | Thông tin |
|------------|-----------|
| Zabbix Server | Ubuntu 24.04 LTS – 192.168.136.154 |
| Phiên bản | Zabbix 7.0 LTS |
| Database | MySQL 8.0 |
| Web Server | Apache2 + PHP |

## Architecture

```
Zabbix Server (192.168.136.154)
├── MySQL Database (zabbix DB)
├── Apache2 + PHP Frontend
└── Zabbix Agent (localhost)
        │
        │ port 10051 (active) / 10050 (passive)
        ▼
   Monitored Hosts
   ├── Host 1 - Zabbix Agent
   ├── Host 2 - Zabbix Agent
   └── Host N - (Ansible deployment)
```

## Cài Đặt Zabbix Server

### 1. Thêm repository và cài đặt
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

### 2. Tạo Database
```sql
mysql -uroot -p
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
```

### 3. Import schema và khởi động
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

### 4. Cấu hình Web
Truy cập: `http://<IP>/zabbix/setup.php` và điền thông tin database.

## Thêm Host Giám Sát

### Cách 1: Cài Zabbix Agent thủ công
```bash
# Trên host cần giám sát
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt install zabbix-agent

# Cấu hình /etc/zabbix/zabbix_agentd.conf
Server=192.168.136.154
ServerActive=192.168.136.154
Hostname=<tên host>

systemctl restart zabbix-agent && systemctl enable zabbix-agent
```

### Cách 2: Tự động bằng Ansible (deploy hàng loạt)
```yaml
# install_zabbix.yml
- hosts: linux
  become: yes
  tasks:
    - name: Install Zabbix agent
      apt:
        name: zabbix-agent
        state: present
    - name: Configure agent
      lineinfile:
        path: /etc/zabbix/zabbix_agentd.conf
        regexp: '^Server='
        line: 'Server=192.168.136.154'
    - name: Start agent
      systemd:
        name: zabbix-agent
        state: started
        enabled: yes
```

```bash
ansible-playbook -i hosts install_zabbix.yml
```

**Điều kiện:** Host đích mở port 22 (SSH) và port 10050 (Zabbix agent), chế độ Bridge network.

## Key Learnings

- Zabbix 7.0 yêu cầu `log_bin_trust_function_creators = 1` khi import schema với MySQL binary logging bật
- Ansible giúp deploy agent lên nhiều máy cùng lúc — không cần thao tác tay từng máy
- Template `Linux by Zabbix agent` có sẵn gần 100 metric, không cần tự cấu hình từ đầu

## Tech Stack

`Zabbix 7.0` `Ubuntu 24.04` `MySQL 8.0` `Apache2` `Ansible` `SNMP`
