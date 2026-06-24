# FortiGate 100D Network Configuration

Cấu hình FortiGate 100D cho môi trường văn phòng đa tầng: routing, inter-VLAN, NAT, VIP, firewall policy.

## Device Info

| Thông tin | Giá trị |
|-----------|---------|
| Model | FortiGate 100D |
| Firmware | FortiOS 6.0.18 (build 0549) |
| WAN | PPPoE (wan1) |
| Management IP | 192.168.10.99 |

## Network Topology

```
Internet
    │
    │ WAN1 (PPPoE)
    ▼
FortiGate 100D (192.168.10.99 mgmt)
    │
    ├── Tang1_SW (port1-6)  → 192.168.1.1/24  (Tầng 1)
    ├── Tang2_SW (port7-8)  → 192.168.2.1/24  (Tầng 2)
    ├── Tang3_SW (port9-10) → 192.168.3.1/24  (Tầng 3)
    ├── Tang4_SW (port11-12)→ 192.168.4.1/24  (Tầng 4)
    ├── Tang5_SW (port13-14)→ 192.168.5.1/24  (Tầng 5)
    └── DMZ                 → 10.10.10.1/24
```

## Cấu Hình Chính

### 1. Switch Interface (Per-Floor VLAN)
```
Tang1_SW: port1-6  → 192.168.1.1/24
Tang2_SW: port7-8  → 192.168.2.1/24
Tang3_SW: port9-10 → 192.168.3.1/24
Tang4_SW: port11-12→ 192.168.4.1/24
Tang5_SW: port13-14→ 192.168.5.1/24
```

Mỗi tầng là một subnet riêng, FortiGate làm inter-VLAN router.

### 2. NAT (Internet Access)
```
config firewall policy
    edit 1
        set srcintf "Tang1_SW"
        set dstintf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set nat enable
    next
```
Các tầng khác cấu hình tương tự.

### 3. VIP (Port Forwarding / Virtual IP)
```
config firewall vip
    edit "VIP-WebServer"
        set extintf "wan1"
        set portforward enable
        set extport 80
        set mappedip 10.10.10.10
        set mappedport 80
    next
```

### 4. DMZ Policy
```
config firewall policy
    edit 10
        set srcintf "wan1"
        set dstintf "dmz"
        set dstaddr "VIP-WebServer"
        set action accept
        set service "HTTP" "HTTPS"
    next
```

### 5. Admin Access
```bash
# SSH và HTTPS management trên port 8443 (đổi từ 443 mặc định)
set admin-sport 8443
set admin-scp enable
```

## Firewall Policy Matrix

| Source | Destination | Action | Notes |
|--------|-------------|--------|-------|
| Tang1-5_SW | wan1 | ACCEPT + NAT | Internet access |
| wan1 | DMZ | ACCEPT | Qua VIP only |
| Tang1-5_SW | Tang1-5_SW | ACCEPT | Inter-floor |
| any | mgmt | DENY | Chặn truy cập management |

## Key Learnings

- FortiGate `switch-interface` cho phép gom nhóm port vật lý thành 1 L3 interface — phù hợp cấu hình per-floor
- PPPoE trên wan1: FortiGate tự xử lý dial-up, không cần router riêng
- VIP và port forwarding tách biệt với NAT outbound — cần policy riêng cho inbound traffic
- `admin-sport 8443` tăng bảo mật bằng cách đổi port HTTPS management khỏi 443 mặc định

## Tech Stack

`FortiGate 100D` `FortiOS 6.0` `PPPoE` `Inter-VLAN Routing` `NAT` `VIP` `DMZ` `Firewall Policy`
