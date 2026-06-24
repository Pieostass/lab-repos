# Backup & Restore Oracle Database bằng Commvault

> **Stack:** Ubuntu 24.04 LTS | Oracle XE 21c (21.3.0.0.0) | Commvault 11.40  
> Tài liệu đầy đủ: cài Oracle native, tích hợp Commvault, backup & restore thực tế kèm bảng xử lý lỗi.

---

## 1. Tổng Quan Kiến Trúc

### Tại sao KHÔNG dùng Docker cho Oracle + Commvault?

Khi RMAN chạy backup, nó yêu cầu Oracle server process tự đọc datafile và ghi backup thông qua thư viện SBT (`libobk.so`) của Commvault — thư viện này phải nằm trong tầm nhìn của tiến trình Oracle server.

> **Khi Oracle chạy trong Docker:** tiến trình Oracle nằm trong container (filesystem riêng), thư viện SBT cài ở `/opt/commvault` trên host — hai bên không thấy nhau → **backup thất bại**.

**Kiến trúc đúng:**
- Oracle XE 21c cài **native** trực tiếp trên Ubuntu 24.04
- Commvault iDataAgent cài trên **cùng host** đó
- RMAN có thể gọi thư viện SBT vì cùng filesystem
- Backup/Restore hoạt động bình thường qua giao thức SBT_TAPE

### Thông số môi trường

| Thành phần | Phiên bản | Ghi chú |
|-----------|----------|---------|
| Ubuntu | 24.04 LTS (Noble) | Host OS |
| Oracle Database | XE 21c (21.3.0.0.0) | Express Edition miễn phí |
| Commvault | 11.40.51 | iDataAgent Oracle |
| VM IP | 10.0.90.243 | hieu-oracle-linux |
| RAM | 8 GB | Tối thiểu 4 GB cho Oracle XE |
| Disk | 78 GB | Tối thiểu 20 GB khuyến nghị |

---

## 2. Cài Đặt Oracle XE 21c trên Ubuntu 24.04

> ⚠️ Ubuntu 24.04 **không hỗ trợ** cài Oracle qua package manager chính thức. Cần cài thủ công bằng `rpm2cpio`.

### Bước 1: Cài gói phụ trợ

```bash
sudo apt update
sudo apt install -y alien libaio1t64 unixodbc bc unzip rpm cpio
```

> ⚠️ **Không dùng `libaio1`** (tên cũ) — sẽ báo lỗi `no installation candidate` trên Ubuntu 24.04. Phải dùng `libaio1t64`.

### Bước 2: Tải file RPM Oracle XE 21c

```bash
cd /tmp
wget https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm

# Kiểm tra file tải thành công (~2.2 GB)
ls -lh /tmp/oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm
```

### Bước 3: Tạo user/group Oracle

```bash
sudo groupadd -g 54321 oinstall
sudo groupadd -g 54322 dba
sudo useradd -u 54321 -g oinstall -G dba -m -d /home/oracle -s /bin/bash oracle
sudo passwd oracle   # Đặt password, ví dụ: Abcabc123

# Kiểm tra
id oracle
# Kết quả mong đợi: uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba)
```

### Bước 4: Set kernel parameters

```bash
sudo tee /etc/sysctl.d/98-oracle.conf > /dev/null << 'EOF'
fs.file-max = 6815744
kernel.sem = 10000 32000 10000 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
kernel.msgmni = 2878
EOF
sudo sysctl -p /etc/sysctl.d/98-oracle.conf

sudo tee /etc/security/limits.d/oracle.conf > /dev/null << 'EOF'
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft nproc 16384
oracle hard nproc 16384
oracle soft stack 10240
oracle hard stack 32768
EOF
```

### Bước 5: Giải nén RPM bằng rpm2cpio

> Không dùng `alien` (chậm, mất 30+ phút). Dùng `rpm2cpio` để giải nén trực tiếp.

```bash
cd /
sudo rpm2cpio /tmp/oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm | sudo cpio -idmv

# Fix ownership
sudo chown -R oracle:oinstall /opt/oracle
sudo find /opt/oracle -type d -exec chmod 755 {} \;
```

> ⚠️ Giải nén **từ thư mục `/`** để đường dẫn tuyệt đối của RPM được extract đúng vị trí.

### Bước 6: Fix symlink Oracle Home

```bash
sudo mkdir -p /opt/oracle/homes
sudo ln -s /opt/oracle/product/21c/dbhomeXE /opt/oracle/homes/OraDBHome21cXE
sudo chown -R oracle:oinstall /opt/oracle/homes
```

### Bước 7: Fix thư viện libaio

```bash
# Kiểm tra file thật nằm ở đâu
dpkg -L libaio1t64 | grep '\.so'

# Tạo symlink đúng tên Oracle cần
sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so.1t64.0.2 /usr/lib/x86_64-linux-gnu/libaio.so.1
sudo ldconfig

# Kiểm tra
ldconfig -p | grep libaio
```

### Bước 8: Fix quyền file oradism

```bash
sudo chown root:oinstall /opt/oracle/product/21c/dbhomeXE/bin/oradism
sudo chmod 4750 /opt/oracle/product/21c/dbhomeXE/bin/oradism

# Kiểm tra — phải thấy chữ 's' (setuid)
ls -la /opt/oracle/product/21c/dbhomeXE/bin/oradism
# Kết quả: -rwsr-x--- 1 root oinstall ... oradism
```

### Bước 9: Fix hostname DNS

```bash
# Thêm IP thật (thay bằng IP thật của VM bạn)
echo '10.0.90.243 hieu-oracle-linux' | sudo tee -a /etc/hosts
cat /etc/hosts
```

### Bước 10: Tắt RemoveIPC ⚠️ CỰC KỲ QUAN TRỌNG

> ⚠️ Nếu bỏ qua bước này, Oracle sẽ bị kill sau vài phút do systemd xóa semaphore IPC.

```bash
# Tắt systemd-oomd
sudo systemctl stop systemd-oomd
sudo systemctl disable systemd-oomd

# Tắt RemoveIPC trong logind.conf
sudo sed -i 's/#RemoveIPC=yes/RemoveIPC=no/' /etc/systemd/logind.conf

# Kiểm tra đã sửa chưa
grep RemoveIPC /etc/systemd/logind.conf   # Phải thấy: RemoveIPC=no

# Restart systemd-logind để áp dụng
sudo systemctl restart systemd-logind

# Tắt transparent hugepages
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### Bước 11: Chạy configure tạo database

```bash
sudo /etc/init.d/oracle-xe-21c configure
```

Khi được hỏi password, đặt password **chỉ gồm chữ + số** (ví dụ: `Abcabc123`):
- Không dùng ký tự đặc biệt (`.`, `!`, v.v.)
- Tối thiểu 8 ký tự, có chữ hoa, chữ thường và số
- Password này dùng chung cho SYS, SYSTEM và PDBADMIN

> Thông báo `Database configuration failed` ở cuối là **false alarm** do warning DBMS_QOPATCH, không ảnh hưởng tới database.

### Bước 12: Khởi động database

```bash
sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  export ORACLE_SID=XE
  \$ORACLE_HOME/bin/sqlplus / as sysdba <<'EOF'
STARTUP;
ALTER SYSTEM REGISTER;
EXIT;
EOF
"

# Kiểm tra process đang chạy
ps -ef | grep -i 'xe_pmon' | grep -v grep

# Kiểm tra listener
sudo -u oracle bash -c "export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE; \$ORACLE_HOME/bin/lsnrctl status"
```

> ⚠️ Phải thấy `Service XE has 1 instance(s)... status READY` trong `lsnrctl status` mới tiếp tục.

### Bước 13: Bật ARCHIVELOG và tạo data test

```bash
sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  export ORACLE_SID=XE
  \$ORACLE_HOME/bin/sqlplus / as sysdba <<'EOF'
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ARCHIVE LOG LIST;
EXIT;
EOF
"
# Kết quả mong đợi:
# Database log mode:    Archive Mode
# Automatic archival:   Enabled
```

```bash
# Tạo bảng test trong PDB XEPDB1
sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  \$ORACLE_HOME/bin/sqlplus system/Abcabc123@//localhost:1521/XEPDB1 <<'EOF'
CREATE TABLE test_backup (id NUMBER, note VARCHAR2(100));
INSERT INTO test_backup VALUES (1, 'Du lieu truoc khi backup');
COMMIT;
SELECT * FROM test_backup;
EXIT;
EOF
"
```

### Bước 14: Thêm vào /etc/oratab để tự start

```bash
sudo bash -c "echo 'XE:/opt/oracle/product/21c/dbhomeXE:Y' >> /etc/oratab"
sudo chown oracle:oinstall /etc/oratab
```

---

## 3. Cài Đặt và Cấu Hình Commvault

### 3.1 Push Install Agent lên VM

Từ Commvault Command Center:
- Vào **Manage → Infrastructure → Servers → Add server**
- Nhập IP: `10.0.90.243`, OS Type: Unix, nhập user/password SSH
- Chọn package: **Oracle Database iDataAgent**
- Submit → theo dõi Job Install

### 3.2 Đổi group Commvault sang oinstall ⚠️ BẮT BUỘC

```bash
sudo /opt/commvault/Base/cvpkgchg
# Khi hỏi Unix Group Name: gõ oinstall → Next → YES

sudo /opt/commvault/Base/commvault restart
```

> ⚠️ Nếu bỏ qua bước này sẽ báo lỗi `[18:179] not installed with OS group that belongs to Oracle user`.

### 3.3 Add Oracle Instance trong Commvault

Vào **Protect → Databases → Add database server:**

| Field | Giá trị |
|-------|---------|
| Host name | 10.0.90.243 |
| OS Type | Unix |
| User name | oracle *(Linux user, KHÔNG phải SYS)* |
| Password | Password của Linux user oracle |
| UNIX group | oinstall |
| SSH port | 22 |

Sau đó vào **Add Oracle instance:**

| Field | Giá trị |
|-------|---------|
| Oracle SID | XE |
| Plan | Chọn plan backup đang có |
| Oracle home | /opt/oracle/product/21c/dbhomeXE |
| Connect string - Username | sys *(không có dấu `/`)* |
| Connect string - Password | Abcabc123 |
| Connect string - Service name | XE |
| Advanced > OS user name | oracle |

### 3.4 Cấu hình subclient default

Vào **Subclients → click default → VIEW/EDIT Content:**
- **Bỏ tick** `Archive logs` khỏi Content
- Lý do: backup archive log riêng sẽ gây lỗi `SDTCSLessArchiveUtils` trên cấu hình này

---

## 4. Thực Hiện Backup

### 4.1 Kiểm tra trước khi backup

```bash
# Kiểm tra process Oracle
ps -ef | grep xe_pmon | grep -v grep

# Kiểm tra listener có services
sudo -u oracle bash -c "export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE; \$ORACLE_HOME/bin/lsnrctl status" | grep -E 'Service|supports'
```

> ⚠️ Nếu thấy `The listener supports no services` → database đã chết. Cần startup lại trước khi backup.

```bash
# Startup lại nếu database chết
sudo ipcs -s | awk 'NR>2 {print $2}' | xargs -I{} sudo ipcrm -s {} 2>/dev/null
sudo ipcs -m | awk 'NR>2 {print $2}' | xargs -I{} sudo ipcrm -m {} 2>/dev/null

sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  export ORACLE_SID=XE
  \$ORACLE_HOME/bin/sqlplus / as sysdba <<'EOF'
STARTUP;
ALTER SYSTEM REGISTER;
EXIT;
EOF
"
```

### 4.2 Chạy Full Backup

1. Vào **Protect → Databases** → tìm instance XE
2. Click `...` → **Back up now**
3. Chọn subclient: chỉ tick **default** (KHÔNG tick ArchiveLog)
4. Click SELECT

**Thông số backup thực tế đạt được:**

| Thông số | Giá trị |
|---------|---------|
| Backup type | Online Full |
| Số objects | 5 datafiles |
| Database size | 2.67 GB |
| Thời gian | ~59 giây |
| Throughput | 686 GB/hr |
| Data transferred | 571 MB (sau dedup/compression 79%) |

---

## 5. Thực Hiện Restore

### 5.1 Mô phỏng mất dữ liệu

```bash
sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  \$ORACLE_HOME/bin/sqlplus system/Abcabc123@//localhost:1521/XEPDB1 <<'EOF'
DROP TABLE test_backup;
SELECT * FROM test_backup;  -- Sẽ báo ORA-00942: table or view does not exist
EXIT;
EOF
"
```

### 5.2 Thực hiện Restore từ Commvault

1. Vào **Protect → Databases** → click vào instance XE
2. Click **Configure recovery** hoặc `...` → **Restore**
3. Chọn **Backup content** → tick `XEPDB1` → click **RESTORE**
4. Restore options: **In place** | Recover to: **Most recent backup**
5. Click SUBMIT

**Thông số restore thực tế:**

| Thông số | Giá trị |
|---------|---------|
| Status | Completed |
| Duration | 38 giây |
| No. of successes | 1 |
| Size of application | 516 MB |
| Throughput | 50.39 GB/hr |

### 5.3 Xác nhận restore thành công

```bash
sudo -u oracle bash -c "
  export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
  \$ORACLE_HOME/bin/sqlplus system/Abcabc123@//localhost:1521/XEPDB1 <<'EOF'
SELECT * FROM test_backup;
EXIT;
EOF
"
# Kết quả mong đợi:
#  ID  NOTE
#  1   Du lieu truoc khi backup  ✅
```

---

## 6. Xử Lý Lỗi Thực Tế

| Lỗi | Nguyên nhân | Cách fix |
|-----|-------------|----------|
| `Package 'libaio1' has no installation candidate` | Ubuntu 24.04 đổi tên gói | `sudo apt install -y libaio1t64` + tạo symlink |
| `error while loading shared libraries: libaio.so.1` | Chưa tạo symlink | `sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so.1t64.0.2 /usr/lib/x86_64-linux-gnu/libaio.so.1 && sudo ldconfig` |
| `TNS-04415 No valid IP Address for hieu-oracle-linux` | Oracle Java không resolve hostname | `echo '10.0.90.243 hieu-oracle-linux' | sudo tee -a /etc/hosts` |
| `[FATAL] Incorrect ownership/permissions for oradism` | `chown -R` đã ghi đè setuid bit | `sudo chown root:oinstall .../oradism && sudo chmod 4750 .../oradism` |
| `ORA-27157: OS post/wait facility removed` ⚠️ | systemd-logind xóa semaphore IPC | `sed -i 's/#RemoveIPC=yes/RemoveIPC=no/' /etc/systemd/logind.conf` + restart logind |
| `ORA-12514: TNS listener does not currently know of service` | Database chưa đăng ký với listener | `sqlplus / as sysdba` → `ALTER SYSTEM REGISTER;` |
| `[18:179] not installed with OS group that belongs to Oracle user` | Commvault group sai | `sudo /opt/commvault/Base/cvpkgchg` → nhập `oinstall` |
| `ORA-03113: end-of-file on communication channel` | Database chết khi RMAN kết nối | Kiểm tra `xe_pmon` process, xem alert log, fix semaphore rồi startup lại |
| Backup treo ở 80% - `SDTCSLessArchiveUtils` | Subclient đang backup archive log | Bỏ tick `Archive logs` trong Content của subclient default |

---

## 7. Checklist Nhanh

### Trước khi backup

| # | Kiểm tra | Lệnh |
|---|---------|------|
| 1 | Database đang chạy | `ps -ef | grep xe_pmon` |
| 2 | Listener có services | `lsnrctl status | grep Service` |
| 3 | Commvault service online | Client có chấm xanh trong Command Center |
| 4 | RemoveIPC=no | `grep RemoveIPC /etc/systemd/logind.conf` |
| 5 | Commvault group = oinstall | `ls -la /opt/commvault/Base | grep oinstall` |

### Thông tin quan trọng

| Thông tin | Giá trị |
|----------|---------|
| Oracle SID | XE |
| Oracle Home | /opt/oracle/product/21c/dbhomeXE |
| PDB name | XEPDB1 |
| Listener port | 1521 |
| OS user Oracle | oracle (UID=54321) |
| Oracle group | oinstall (GID=54321) |
| SYS/SYSTEM password | Abcabc123 |
| Alert log | /opt/oracle/diag/rdbms/xe/XE/trace/alert_XE.log |
| VM IP | 10.0.90.243 (hieu-oracle-linux) |

---

*Tài liệu được tổng hợp từ thực nghiệm triển khai thực tế — Oracle XE 21c + Commvault 11.40 trên Ubuntu 24.04 LTS*
