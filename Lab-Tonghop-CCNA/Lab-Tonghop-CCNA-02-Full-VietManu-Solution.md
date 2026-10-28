# Lab Tổng Hợp CCNA 02 — Full: VietManu Manufacturing — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-02-Full-VietManu.md`

---

## 1. R-HQ-1

```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 172.20.0.1 255.255.255.224
 ipv6 address 2001:DB8:<4-hex>:1::1/64
 ip nat inside
 vrrp 10 ip 172.20.0.3
 vrrp 10 priority 110
 vrrp 10 preempt

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 172.20.0.33 255.255.255.224
! KHONG "ip nat inside" o day - SANXUAT khong duoc phep NAT ra Internet duoi bat ky hinh thuc nao

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 172.20.0.65 255.255.255.248
 ip nat inside

interface Serial0/1/0
 ip address 172.20.0.73 255.255.255.252
 ip nat inside
interface Serial0/1/1
 ip address 172.20.0.77 255.255.255.252
 ip nat inside
interface Serial0/0/1
 ip address 172.20.0.81 255.255.255.252
 ip nat outside
interface Serial0/0/2
 ip address 198.51.100.5 255.255.255.252
 ip nat outside

! IPv6 (Task 2): thay <4-hex> bang 4 hex da chon cho 2001:db8:<4-hex>::/48
ipv6 unicast-routing

! --- Task 7: EIGRP voi Warehouse + quang ba default route ---
ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
route-map DEFAULT-TO-EIGRP permit 10
 match ip address prefix-list DEFAULT-ROUTE

router eigrp 200
 network 172.20.0.76 0.0.0.3
 network 172.20.0.0 0.0.0.31
 network 172.20.0.64 0.0.0.7
 ! (khong co network cho SANXUAT 172.20.0.32/27 - OT khong duoc quang ba)
 redistribute static route-map DEFAULT-TO-EIGRP metric 1000000 1 255 1 1500
 no auto-summary

! --- Task 8-9: Static route Workshop + default route ISP + failover ---
ip route 172.20.1.0 255.255.255.240 172.20.0.74
ip sla 1
 icmp-echo 172.20.0.82 source-interface Serial0/0/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
ip route 0.0.0.0 0.0.0.0 172.20.0.82 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.6 200

! --- Task 10: DHCP (chi VANPHONG) ---
ip dhcp excluded-address 172.20.0.1 172.20.0.5
ip dhcp pool VANPHONG_POOL
 network 172.20.0.0 255.255.255.224
 default-router 172.20.0.3

! --- Task 11: NAT (loai tru SANXUAT) ---
! NAT_ACL (extended) thay cho access-list 1: co deny VPN pool 172.20.50.0/24 de goi tra ve remote user KHONG bi NAT
! (neu bi NAT se khong khop ACL 101/crypto map nen khong vao tunnel)
ip access-list extended NAT_ACL
 deny ip 172.20.0.0 0.0.0.31 172.20.50.0 0.0.0.255
 permit ip 172.20.0.0 0.0.0.31 any
 permit ip 172.20.0.64 0.0.0.7 any
 permit ip 172.20.1.0 0.0.0.15 any
 permit ip 172.20.2.0 0.0.0.15 any
! KHONG co dong access-list nao cho 172.20.0.32 0.0.0.31 (SANXUAT)
! PAT theo interface ra (ISP-A Se0/0/1 hoac ISP-B Se0/0/2) de failover van NAT duoc
route-map NAT_ISPA permit 10
 match ip address NAT_ACL
 match interface Serial0/0/1
route-map NAT_ISPB permit 10
 match ip address NAT_ACL
 match interface Serial0/0/2
ip nat inside source route-map NAT_ISPA interface Serial0/0/1 overload
ip nat inside source route-map NAT_ISPB interface Serial0/0/2 overload

! --- Task 12: ACL cach ly SanXuat <-> VanPhong (2 chieu), van cho SanXuat toi IT_SERVER (syslog) ---
ip access-list extended SANXUAT_ISOLATION
 deny ip 172.20.0.32 0.0.0.31 172.20.0.0 0.0.0.31
! SANXUAT chi duoc toi IT_SERVER (syslog/AAA) va gateway cua chinh no (NTP); moi dich khac - ke ca Internet/RFC1918 - bi chan RIENG bang ACL
! (khong chi dua vao viec thieu NAT). Dong "permit ip any any" phia duoi khong con tac dung voi traffic nguon SANXUAT.
 permit ip 172.20.0.32 0.0.0.31 172.20.0.64 0.0.0.7
 permit ip 172.20.0.32 0.0.0.31 172.20.0.32 0.0.0.31
 deny ip 172.20.0.32 0.0.0.31 any
 permit ip any any
interface GigabitEthernet0/0.20
 ip access-group SANXUAT_ISOLATION in

ip access-list extended VANPHONG_ISOLATION
 deny ip 172.20.0.0 0.0.0.31 172.20.0.32 0.0.0.31
 permit ip any any
interface GigabitEthernet0/0.10
 ip access-group VANPHONG_ISOLATION in

! --- Task 13: NTP/Syslog ---
ntp master 3
service timestamps log datetime msec
logging host 172.20.0.69

! --- Task 16: VPN Remote Access (chi vao VANPHONG) ---
aaa new-model
aaa authentication login VPN_XAUTH local
aaa authorization network VPN_GROUP local
username remote01 secret Cisco@12345
ip local pool VPN_POOL 172.20.50.10 172.20.50.20
access-list 101 permit ip 172.20.0.0 0.0.0.31 172.20.50.0 0.0.0.255
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp client configuration group VIETMANU-VPN
 key VmVpn2026
 pool VPN_POOL
 acl 101
crypto ipsec transform-set VPN_TS esp-aes esp-sha-hmac
crypto dynamic-map VPN_DYNMAP 10
 set transform-set VPN_TS
 reverse-route
crypto map VPN_MAP client authentication list VPN_XAUTH
crypto map VPN_MAP isakmp authorization list VPN_GROUP
crypto map VPN_MAP client configuration address respond
crypto map VPN_MAP 10 ipsec-isakmp dynamic VPN_DYNMAP
interface Serial0/0/1
 crypto map VPN_MAP

! --- Task 19: AAA quan tri thiet bi ---
tacacs server SRV-AAA
 address ipv4 172.20.0.67
 key TacacsKeyVM2026
aaa authentication login VTY_AUTH group tacacs+ local
username backupadmin secret Local@BackupVM
line vty 0 4
 login authentication VTY_AUTH
```

## 2. R-HQ-2 (VRRP Backup)

> **Lưu ý cập nhật:** đây là cấu hình **một phần** của R-HQ-2 (chỉ subinterface VLAN 10 + VRRP, không có EIGRP/WAN/VLAN 20, 30). Kịch bản Task 14 chỉ chứng minh VRRP chuyển Master cho VLAN VANPHONG; failover toàn phần cần bổ sung EIGRP, subinterface còn lại và đường WAN trên R-HQ-2.

```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 172.20.0.2 255.255.255.224
 vrrp 10 ip 172.20.0.3
 vrrp 10 priority 100

ntp server 172.20.0.1
logging host 172.20.0.69
```

## 3. R-WAREHOUSE (EIGRP)

```
interface Serial0/0/0
 ip address 172.20.0.78 255.255.255.252
interface GigabitEthernet0/0
 ip address 172.20.2.1 255.255.255.240

router eigrp 200
 network 172.20.2.0 0.0.0.15
 network 172.20.0.76 0.0.0.3
 no auto-summary

ntp server 172.20.0.77
logging host 172.20.0.69
```

## 4. R-WORKSHOP (chỉ static route)

```
interface Serial0/0/0
 ip address 172.20.0.74 255.255.255.252
interface GigabitEthernet0/0
 ip address 172.20.1.1 255.255.255.240

! Chi 1 static route duy nhat, tro moi thu ve HQ
ip route 0.0.0.0 0.0.0.0 172.20.0.73

ntp server 172.20.0.73
logging host 172.20.0.69
```

## 4a. SW-DIST-VM (VLAN, Trunk, EtherChannel, STP)

```
vlan 10
 name VANPHONG
vlan 20
 name SANXUAT
vlan 30
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED

! EtherChannel LACP xuong SW-ACC-VANPHONG (2 day; cong fa0/1-2 theo quy uoc Lab 01)
interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999

! Trunk xuong SW-ACC-SANXUAT (vi du cong fa0/3) - native 999, KHONG gan host vao VLAN 999
interface fa0/3
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999

! Cac trunk toi R-HQ-1/R-HQ-2/WLC-VM/ASA-VM dung cung native 999 va allowed vlan tuong ung
spanning-tree vlan 10,20,30 priority 4096
```

## 5. SW-ACC-VANPHONG (Layer 2 tiêu chuẩn)

> **Lưu ý cập nhật:** bảng địa chỉ chưa cấp IP quản trị cho các switch. Để TACACS+/syslog/NTP chạy được từ switch cần thêm 1 SVI quản trị (ví dụ trong VLAN 30) `interface vlan 30` / `ip address <IP còn trống trong 172.20.0.64/29> 255.255.255.248` và `ip default-gateway 172.20.0.65`.

```
! Uplink EtherChannel (cong fa0/1-2 theo quy uoc Lab 01) - phai TRUST de DHCP Snooping/DAI khong chan DHCP offer tu R-HQ-1 va ARP tu gateway
interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999
 ip dhcp snooping trust
 ip arp inspection trust

interface range fa0/5-20
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

ip dhcp snooping
ip dhcp snooping vlan 10
ip arp inspection vlan 10
```

## 6. SW-ACC-SANXUAT (Layer 2 nghiêm ngặt hơn — thiết bị OT)

```
! Uplink len SW-DIST-VM (vi du fa0/1) la trunk, KHONG nam trong range access
interface fa0/1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999

interface range fa0/5-10
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```
Không cấu hình DHCP Snooping cho VLAN 20 vì VLAN này không có DHCP — cấu hình sẽ vô nghĩa (không có gì để snoop).

## 7. WLC-VM

```
config interface address management 172.20.0.68 255.255.255.248 172.20.0.65
config interface create VANPHONG-DATA 10
config interface address dynamic-interface VANPHONG-DATA 172.20.0.4 255.255.255.224 172.20.0.3
config interface vlan dynamic-interface VANPHONG-DATA 10
config wlan create 1 VIETMANU-STAFF-WIFI VIETMANU-STAFF-WIFI
config wlan interface 1 VANPHONG-DATA
config wlan security wpa wpa2 enable 1
config wlan security wpa akm psk enable 1
config wlan security wpa akm psk set-key ascii Staff@VM2026 1
config wlan enable 1
```

## 8. ASA-VM (DMZ cho ERP, chỉ mở 443)

> **Lưu ý cập nhật:** (1) Từ ASA 8.3 trở lên, ACL chiều `outside` phải tham chiếu **địa chỉ thật** của server (`172.21.0.10`), không dùng IP mapped (`200.150.60.10`). (2) IP mapped `.10` nằm ngoài subnet `/30` của link ASA–ISP-C (`200.150.60.0/30`): ISP-C cần có route `200.150.60.10/32` qua `200.150.60.1` (ASA outside) hoặc mở rộng link lên /28. (3) ASA cũng cần `route inside` về mạng nội bộ qua R-HQ-1 (`172.20.0.65`) và các interface phải `no shutdown`; các dòng này chưa có trong cấu hình mẫu bên dưới.

```
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 200.150.60.1 255.255.255.252
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 172.20.0.66 255.255.255.248
interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 172.21.0.1 255.255.255.0

object network SRV-ERP-OBJ
 host 172.21.0.10
 nat (dmz,outside) static 200.150.60.10

! ASA 8.3+: ACL ngoai phai tham chieu IP THAT cua server (172.21.0.10), khong dung IP mapped 200.150.60.10
access-list OUTSIDE_IN extended permit tcp any host 172.21.0.10 eq 443
access-group OUTSIDE_IN in interface outside
route outside 0.0.0.0 0.0.0.0 200.150.60.2
```

---

## 8a. Automation (Task 20)

```python
# backup_devices.py - Netmiko backup show running-config
import csv, datetime, pathlib
from netmiko import ConnectHandler

out = pathlib.Path("backup"); out.mkdir(exist_ok=True)
for d in csv.DictReader(open("inventory.csv")):   # cot: host,device_type,username,password
    try:
        with ConnectHandler(**d) as conn:
            cfg = conn.send_command("show running-config")
        (out / f"{d['host']}_{datetime.date.today()}.cfg").write_text(cfg)
    except Exception as e:
        print(d["host"], "FAIL:", e)
```

```yaml
# deploy_xuong_moi.yml - chi nham nhom xuong_moi, khong dung Workshop/Warehouse (IP lay tu host_vars)
- hosts: xuong_moi
  gather_facts: false
  tasks:
    - cisco.ios.ios_l3_interfaces:
        config:
          - name: GigabitEthernet0/0
            ipv4: [{address: "{{ lan_ip_cidr }}"}]
        state: merged
    - cisco.ios.ios_static_routes:
        config:
          - address_families:
              - afi: ipv4
                routes:
                  - dest: 0.0.0.0/0
                    next_hops: [{forward_router_address: "{{ hq_next_hop }}"}]
        state: merged
```

## 9. Output verify tổng hợp

| Task | Lệnh | Kết quả mong đợi |
|---|---|---|
| 7 | `show ip eigrp neighbors` (R-WAREHOUSE) | Neighbor `172.20.0.77` |
| 9 | `show ip route \| include D*EX` (R-WAREHOUSE) | Có `0.0.0.0/0` |
| 11 | `ping 8.8.8.8` từ PC VLAN SANXUAT | **Phải timeout** |
| 12 | `ping` chéo SanXuat↔VanPhong | Timeout cả 2 chiều |
| 14 | `show vrrp brief` khi tắt R-HQ-1 | R-HQ-2 chuyển Master |
| 18 | `show port-security interface` (SW-ACC-SANXUAT) | Violation Mode: Shutdown, Max 1 |

## 10. Giải thích các điểm quan trọng / dễ sai

- **EIGRP không có `default-information originate` như OSPF** — phải dùng `redistribute static` (kèm route-map lọc đúng chỉ default route, tránh redistribute nhầm các static route khác) — đây là khác biệt quan trọng nhất giữa 2 giao thức mà học viên hay lẫn lộn khi chuyển từ Lab 01 (OSPF) sang Lab này (EIGRP).
- **Không đánh `ip nat inside` trên interface SANXUAT** là lớp phòng thủ thứ 2 — dù access-list NAT có vô tình viết sai bao trùm cả SanXuat, traffic từ interface không được đánh dấu `nat inside` vẫn không bao giờ được dịch địa chỉ.
- **Violation Mode `shutdown` khác `restrict`**: `restrict` vẫn drop gói vi phạm nhưng port sống tiếp; `shutdown` đưa port vào trạng thái `err-disabled` cần can thiệp thủ công (`shutdown` / `no shutdown`) mới lên lại — đúng chủ đích cho mạng OT nơi 1 thiết bị lạ xuất hiện là sự kiện nghiêm trọng cần con người xác nhận trước khi cho kết nối lại.
- **Workshop chỉ có 1 static default route duy nhất** — đây là thiết kế hợp lý cho site nhỏ chỉ có 1 đường ra, không cần thêm route cụ thể cho từng subnet khác vì mọi traffic đều đi qua đúng 1 next-hop (R-HQ-1) nên default route đã đủ.
