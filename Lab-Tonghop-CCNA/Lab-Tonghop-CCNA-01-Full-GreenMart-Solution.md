# Lab Tổng Hợp CCNA — Full: GreenMart Retail — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-01-Full-GreenMart.md`
> Cấu hình gộp theo từng thiết bị (giống dạng "final lab" thực tế) — đối chiếu ngược lại số Task ở mục 7.

---

## 1. R-HQ-1 (router-on-a-stick, HSRP Active, OSPF, NAT, ACL, VPN, WAN failover)

```
interface GigabitEthernet0/0
 no ip address
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.1 255.255.255.224
 ipv6 address 2001:DB8:AABB:1::1/64
 standby 10 ip 10.10.0.3
 standby 10 priority 110
 standby 10 preempt

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.10.0.33 255.255.255.224
 ipv6 address 2001:DB8:AABB:2::1/64

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 10.10.0.65 255.255.255.248

interface Serial0/1/0
 ip address 10.10.0.73 255.255.255.252
interface Serial0/1/1
 ip address 10.10.0.77 255.255.255.252
interface Serial0/0/1
 ip address 10.10.0.81 255.255.255.252
interface Serial0/0/2
 ip address 198.51.100.1 255.255.255.252
 ip nat outside

! --- Task 7-8: OSPF + default route ---
router ospf 1
 network 10.10.0.0 0.0.0.31 area 0
 network 10.10.0.32 0.0.0.31 area 0
 network 10.10.0.64 0.0.0.7 area 0
 network 10.10.0.72 0.0.0.3 area 0
 network 10.10.0.76 0.0.0.3 area 0
 default-information originate

ip sla 1
 icmp-echo 10.10.0.82 source-interface Serial0/0/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
ip route 0.0.0.0 0.0.0.0 10.10.0.82 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.2 200

! --- Task 9: DHCP ---
ip dhcp excluded-address 10.10.0.1 10.10.0.5
ip dhcp excluded-address 10.10.0.33 10.10.0.35
ip dhcp excluded-address 10.10.0.60
! 10.10.0.60 = dynamic interface WAREHOUSE-DATA cua WLC; dns-server 8.8.8.8 la DNS cong cong (thay bang DNS noi bo neu co)
ip dhcp pool OFFICE_POOL
 network 10.10.0.0 255.255.255.224
 dns-server 8.8.8.8
 default-router 10.10.0.3
ip dhcp pool WAREHOUSE_POOL
 network 10.10.0.32 255.255.255.224
 dns-server 8.8.8.8
 default-router 10.10.0.33

! --- Task 10: NAT ---
! (access-list 1 chuan cu duoc thay bang extended ACL NAT_ACL ben duoi)
! NAT_ACL: LOAI TRU traffic tra ve VPN pool (10.10.50.0/24) khoi NAT - neu khong, goi tra ve cho remote user bi doi sang IP outside
! truoc khi vao crypto map, khong khop ACL 101 nen khong duoc ma hoa/gui qua tunnel
ip access-list extended NAT_ACL
 deny ip 10.10.0.0 0.0.255.255 10.10.50.0 0.0.0.255
 permit ip 10.10.0.0 0.0.255.255 any
! Hai route-map: PAT theo dung interface ra (ISP-A Se0/0/1 hoac ISP-B Se0/0/2) de failover van NAT duoc
route-map NAT_ISPA permit 10
 match ip address NAT_ACL
 match interface Serial0/0/1
route-map NAT_ISPB permit 10
 match ip address NAT_ACL
 match interface Serial0/0/2
ip nat inside source route-map NAT_ISPA interface Serial0/0/1 overload
ip nat inside source route-map NAT_ISPB interface Serial0/0/2 overload
interface Serial0/0/1
 ip nat outside
interface GigabitEthernet0/0.10
 ip nat inside
interface GigabitEthernet0/0.20
 ip nat inside
interface GigabitEthernet0/0.30
 ip nat inside
interface Serial0/1/0
 ip nat inside
interface Serial0/1/1
 ip nat inside

! --- Task 11: ACL cach ly Office <-> Warehouse (2 chieu) ---
ip access-list extended WAREHOUSE_ISOLATION
 deny ip 10.10.0.32 0.0.0.31 10.10.0.0 0.0.0.31
 permit ip any any
interface GigabitEthernet0/0.20
 ip access-group WAREHOUSE_ISOLATION in

ip access-list extended OFFICE_ISOLATION
 deny ip 10.10.0.0 0.0.0.31 10.10.0.32 0.0.0.31
 permit ip any any
interface GigabitEthernet0/0.10
 ip access-group OFFICE_ISOLATION in

! --- Task 12: NTP + Syslog ---
ntp master 3
service timestamps log datetime msec
logging host 10.10.0.69

! --- Task 15: VPN Remote Access (Easy VPN Server) ---
aaa new-model
aaa authentication login VPN_XAUTH local
aaa authorization network VPN_GROUP local
username remote01 secret Cisco@12345
ip local pool VPN_POOL 10.10.50.10 10.10.50.20
access-list 101 permit ip 10.10.0.0 0.0.0.31 10.10.50.0 0.0.0.255
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp client configuration group GREENMART-VPN
 key GmVpn2026
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

! --- Task 18: AAA quan tri thiet bi ---
tacacs server SRV-AAA
 address ipv4 10.10.0.67
 key TacacsKeyGM2026
aaa authentication login VTY_AUTH group tacacs+ local
username backupadmin secret Local@BackupGM
line vty 0 4
 login authentication VTY_AUTH
```

## 2. R-HQ-2 (HSRP Standby)

> **Lưu ý cập nhật:** đây là cấu hình **một phần** của R-HQ-2 (chỉ subinterface VLAN 10 + HSRP, không có OSPF/WAN/VLAN 20, 30). Vì vậy kịch bản Task 13 chỉ chứng minh HSRP chuyển Active cho VLAN OFFICE; muốn failover toàn phần (ra Internet, tới Store) cần bổ sung thêm OSPF, subinterface VLAN 20/30 và đường WAN trên R-HQ-2.

```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.10.0.2 255.255.255.224
 standby 10 ip 10.10.0.3
 standby 10 priority 100
 standby 10 preempt

ntp server 10.10.0.1
logging host 10.10.0.69
```

## 3. SW-DIST-GM (VLAN, EtherChannel, STP)

```
vlan 10
 name OFFICE
vlan 20
 name WAREHOUSE
vlan 30
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED

interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999

spanning-tree vlan 10,20,30 priority 4096
```

## 4. SW-ACC-GM (VLAN, Port Security, DHCP Snooping, DAI)

```
vlan 10
 name OFFICE
vlan 20
 name WAREHOUSE
vlan 30
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED

interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,999
 ip dhcp snooping trust
 ip arp inspection trust

! fa0/5-12: VLAN 10 (OFFICE); fa0/13-20: VLAN 20 (WAREHOUSE) - cung cau hinh port-security
interface range fa0/5-12
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
interface range fa0/13-20
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20

tacacs server SRV-AAA
 address ipv4 10.10.0.67
 key TacacsKeyGM2026
aaa new-model
aaa authentication login VTY_AUTH group tacacs+ local
username backupadmin secret Local@BackupGM
line vty 0 4
 login authentication VTY_AUTH
```

## 5. R-STOREA / R-STOREB (OSPF)

```
! R-STOREA
interface Serial0/0/0
 ip address 10.10.0.74 255.255.255.252
interface GigabitEthernet0/0
 ip address 10.10.1.1 255.255.255.240

router ospf 1
 network 10.10.1.0 0.0.0.15 area 0
 network 10.10.0.72 0.0.0.3 area 0

ntp server 10.10.0.73
logging host 10.10.0.69
```
R-STOREB tương tự, đổi sang `10.10.0.78`, LAN `10.10.2.0/28`, network OSPF `10.10.0.76 0.0.0.3`.

## 6. WLC-GM (Wireless)

```
config interface address management 10.10.0.68 255.255.255.248 10.10.0.65
config interface create WAREHOUSE-DATA 20
config interface address dynamic-interface WAREHOUSE-DATA 10.10.0.60 255.255.255.224 10.10.0.33
config interface vlan dynamic-interface WAREHOUSE-DATA 20
config wlan create 1 GREENMART-SCANNER GREENMART-SCANNER
config wlan interface 1 WAREHOUSE-DATA
config wlan security wpa wpa2 enable 1
config wlan security wpa akm psk enable 1
config wlan security wpa akm psk set-key ascii Scanner@GM2026 1
config wlan enable 1
```

## 7. ASA-GM (Firewall + DMZ)

> **Lưu ý cập nhật:** (1) Từ ASA 8.3 trở lên, ACL chiều `outside` phải tham chiếu **địa chỉ thật** của server (`172.20.0.10`), không dùng IP mapped (`200.100.50.10`) vì ASA đã bỏ NAT trước khi kiểm tra ACL. (2) IP mapped `.10` nằm ngoài subnet `/30` của link ASA–ISP-C (`200.100.50.0/30`): ISP-C cần có route `200.100.50.10/32` qua `200.100.50.1` (ASA outside) hoặc mở rộng link lên /28. (3) ASA cần `route inside` về `10.10.0.0/16` qua R-HQ-1 (`10.10.0.65`) và các interface phải `no shutdown`.

```
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 200.100.50.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.10.0.66 255.255.255.248
 no shutdown
interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 172.20.0.1 255.255.255.0
 no shutdown

object network SRV-WEB-OBJ
 host 172.20.0.10
 nat (dmz,outside) static 200.100.50.10

! ASA 8.3+: ACL ngoai phai tham chieu IP THAT cua server (172.20.0.10), khong dung IP mapped 200.100.50.10
access-list OUTSIDE_IN extended permit tcp any host 172.20.0.10 eq 80
access-list OUTSIDE_IN extended permit tcp any host 172.20.0.10 eq 443
access-group OUTSIDE_IN in interface outside
route outside 0.0.0.0 0.0.0.0 200.100.50.2
route inside 10.10.0.0 255.255.0.0 10.10.0.65
```

## 8. Automation (Task 20)

`backup_devices.py` (Netmiko) và `deploy_store_c.yml` (Ansible, `hosts: store_c`, dùng module `cisco.ios.ios_l3_interfaces` + `cisco.ios.ios_ospfv2`) — nội dung đầy đủ giữ nguyên theo mẫu đã xây dựng ở Mini Project Module 07 (`inventory.csv`, `try/except` từng thiết bị, Ansible Vault cho credential), chỉ đổi IP mục tiêu sang thiết bị của GreenMart.

---

## 9. Output verify tổng hợp (theo từng Task)

| Task | Lệnh | Kết quả mong đợi |
|---|---|---|
| 6 | `show ip interface brief` | 3 subinterface `up/up` đúng IP |
| 7 | `show ip route ospf` (Store A) | Thấy đủ `10.10.0.0/27`, `10.10.0.32/27` |
| 8 | `show ip route \| include ^O\*E2` | Có `0.0.0.0/0` học qua OSPF |
| 10 | `show ip nat translations` | Có bản ghi khi ping ra ngoài |
| 11 | `ping` chéo VLAN | Warehouse→Office fail, Warehouse→Internet OK |
| 13 | `show standby brief` (tắt R-HQ-1) | R-HQ-2 chuyển Active, VIP không đổi |
| 19 | `show nat` (ASA) | Static NAT đúng `200.100.50.10` |

## 10. Giải thích các điểm quan trọng / dễ sai

- **1 topology dùng chung cho toàn bộ 20 Task** nghĩa là lỗi ở Task đầu (vd sai VLAN Task 3) sẽ lan sang mọi Task sau — luôn verify xong 1 Task mới sang Task kế tiếp, đừng cấu hình dồn hết rồi mới test.
- **Task 8 phụ thuộc Task 7** — quên `default-information originate` là lỗi tái diễn nhiều lần trong khóa (đã gặp ở Mini Project Module 03 và Lab Troubleshooting CCNA-04) vì đây là điểm dễ quên nhất khi làm OSPF.
- **Task 11 cần 2 ACL** (không phải 1) cho đúng 2 chiều — lỗi phổ biến giống Lab CCNA-01 (bản cũ) đã từng nhắc.
- **ACL trên ASA (`OUTSIDE_IN`) phải đặt trước khi test Task 19** — thiếu bước này, static NAT vẫn "có" nhưng traffic từ ngoài bị chặn bởi ACL mặc định (deny toàn bộ) của ASA.
- **Automation (Task 20) không sửa cấu hình Store A/B đang chạy** — đúng nguyên tắc `hosts: store_c` đã học ở Mini Project Module 07, tránh vô tình đẩy cấu hình sai sang site đang hoạt động ổn định.
