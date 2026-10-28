# Lab Tổng Hợp CCNA 03 — Full: SunCare Hospital Group — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-03-Full-SunCare.md`

---

## 1. R-HQ-1

```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.50.1 255.255.255.224
 ipv6 address 2001:DB8:<4-hex>:1::1/64
 ip nat inside

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.50.33 255.255.255.224
 glbp 10 ip 192.168.50.35
 glbp 10 priority 110
 glbp 10 preempt
! KHONG "ip nat inside" - NOITRU khong duoc ra Internet

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.50.65 255.255.255.248
! KHONG "ip nat inside" - IT_SERVER (ho so benh an) khong duoc ra Internet truc tiep

interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.50.97 255.255.255.224
 ip nat inside

interface Serial0/1/0
 ip address 192.168.50.73 255.255.255.252
 ip nat inside
 ip ospf message-digest-key 1 md5 SunCareOSPFKey2026
interface Serial0/0/1
 ip address 192.168.50.77 255.255.255.252
 ip nat outside
interface Serial0/0/2
 ip address 198.51.100.9 255.255.255.252
 ip nat outside

! --- Task 7-8: OSPF co xac thuc MD5 + default route ---
router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest
 network 192.168.50.0 0.0.0.31 area 0
 network 192.168.50.72 0.0.0.3 area 0
 default-information originate
! Chu y: KHONG co dong "network" cho NOITRU (192.168.50.32) hay IT_SERVER (192.168.50.64)
! -> Clinic khong hoc duoc route CU THE toi 2 vung nay, NHUNG van co default route ve HQ va R-HQ-1 khong co ACL nen chua phai la cach ly
! -> ACL CLINIC_IN ben duoi moi that su chan Clinic -> NOITRU/IT_SERVER (chi mo syslog .69 va AAA .67 de log/TACACS+ van chay)
ip access-list extended CLINIC_IN
 permit ip 192.168.51.0 0.0.0.15 host 192.168.50.67
 permit ip 192.168.51.0 0.0.0.15 host 192.168.50.69
 deny ip 192.168.51.0 0.0.0.15 192.168.50.32 0.0.0.31
 deny ip 192.168.51.0 0.0.0.15 192.168.50.64 0.0.0.7
 permit ip any any
interface Serial0/1/0
 ip access-group CLINIC_IN in

! IPv6 (Task 2): thay <4-hex> bang 4 hex da chon cho 2001:db8:<4-hex>::/48
ipv6 unicast-routing

! --- Task 9: DHCP (chi NGOAITRU va GUEST) ---
ip dhcp excluded-address 192.168.50.1 192.168.50.5
ip dhcp pool NGOAITRU_POOL
 network 192.168.50.0 255.255.255.224
 default-router 192.168.50.1
ip dhcp excluded-address 192.168.50.97 192.168.50.100
! .97 = gateway, .100 = WLC GUEST-DATA
ip dhcp pool GUEST_POOL
 network 192.168.50.96 255.255.255.224
 default-router 192.168.50.97
 dns-server 8.8.8.8

! --- Task 10: NAT (loai tru NOITRU va IT_SERVER) ---
! NAT_ACL (extended) thay cho access-list 1: co deny VPN pool 192.168.60.0/24 de goi tra ve remote user KHONG bi NAT
! (neu bi NAT se khong khop ACL 101/crypto map nen khong vao tunnel)
ip access-list extended NAT_ACL
 deny ip 192.168.50.0 0.0.0.31 192.168.60.0 0.0.0.255
 permit ip 192.168.50.0 0.0.0.31 any
 permit ip 192.168.50.96 0.0.0.31 any
 permit ip 192.168.51.0 0.0.0.15 any
! PAT theo interface ra (ISP-A Se0/0/1 hoac ISP-B Se0/0/2) de failover van NAT duoc
route-map NAT_ISPA permit 10
 match ip address NAT_ACL
 match interface Serial0/0/1
route-map NAT_ISPB permit 10
 match ip address NAT_ACL
 match interface Serial0/0/2
ip nat inside source route-map NAT_ISPA interface Serial0/0/1 overload
ip nat inside source route-map NAT_ISPB interface Serial0/0/2 overload

! --- Task 11: ACL cach ly ---
ip access-list extended GUEST_ISOLATION
 deny ip 192.168.50.96 0.0.0.31 192.168.50.0 0.0.0.31
 deny ip 192.168.50.96 0.0.0.31 192.168.50.32 0.0.0.31
 deny ip 192.168.50.96 0.0.0.31 192.168.50.64 0.0.0.7
! Chan toan bo dai RFC1918 de khach khong toi duoc Clinic (192.168.51.0), DMZ (192.168.61.0), VPN pool (192.168.60.0)... chi cho ra Internet
 deny ip 192.168.50.96 0.0.0.31 192.168.0.0 0.0.255.255
 deny ip 192.168.50.96 0.0.0.31 10.0.0.0 0.255.255.255
 deny ip 192.168.50.96 0.0.0.31 172.16.0.0 0.15.255.255
 permit ip any any
interface GigabitEthernet0/0.40
 ip access-group GUEST_ISOLATION in

ip access-list extended NOITRU_ISOLATION
 deny ip 192.168.50.32 0.0.0.31 192.168.50.0 0.0.0.31
! NOITRU chi duoc toi IT_SERVER (ho so/syslog/AAA) va gateway cua chinh no (NTP); moi dich khac - ke ca Internet/RFC1918 - bi chan RIENG
! (khong chi dua vao viec thieu NAT). Dong "permit ip any any" phia duoi khong con tac dung voi traffic nguon NOITRU.
 permit ip 192.168.50.32 0.0.0.31 192.168.50.64 0.0.0.7
 permit ip 192.168.50.32 0.0.0.31 192.168.50.32 0.0.0.31
 deny ip 192.168.50.32 0.0.0.31 any
 permit ip any any
interface GigabitEthernet0/0.20
 ip access-group NOITRU_ISOLATION in

ip access-list extended NGOAITRU_ISOLATION
 deny ip 192.168.50.0 0.0.0.31 192.168.50.32 0.0.0.31
 permit ip any any
interface GigabitEthernet0/0.10
 ip access-group NGOAITRU_ISOLATION in

! --- Task 12: NTP/Syslog ---
ntp master 3
service timestamps log datetime msec
logging host 192.168.50.69

! --- Task 14: WAN failover ---
ip sla 1
 icmp-echo 192.168.50.78 source-interface Serial0/0/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
ip route 0.0.0.0 0.0.0.0 192.168.50.78 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.10 200

! --- Task 15: VPN Remote Access (chi vao NGOAITRU) ---
aaa new-model
aaa authentication login VPN_XAUTH local
aaa authorization network VPN_GROUP local
username doctor01 secret Doctor@12345
ip local pool VPN_POOL 192.168.60.10 192.168.60.20
access-list 101 permit ip 192.168.50.0 0.0.0.31 192.168.60.0 0.0.0.255
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp client configuration group SUNCARE-VPN
 key ScVpn2026
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

! --- Task 18: AAA ---
tacacs server SRV-AAA
 address ipv4 192.168.50.67
 key TacacsKeySC2026
aaa authentication login VTY_AUTH group tacacs+ local
username backupadmin secret Local@BackupSC
line vty 0 4
 login authentication VTY_AUTH
```

## 2. R-HQ-2 (GLBP thành viên phụ)

> **Lưu ý cập nhật:** đây là cấu hình **một phần** của R-HQ-2 (chỉ subinterface VLAN 20 + GLBP, chưa có route/OSPF/WAN). Vì GLBP chia tải round-robin, khoảng một nửa client NOITRU được AVF của R-HQ-2 phục vụ sẽ bị **black-hole** nếu R-HQ-2 không có route ra ngoài. Cần bổ sung OSPF (hoặc default route) và subinterface còn lại trên R-HQ-2 để lab hoạt động đầy đủ.

```
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.50.34 255.255.255.224
 glbp 10 ip 192.168.50.35
 glbp 10 priority 100

ntp server 192.168.50.1
logging host 192.168.50.69
```

## 3. R-CLINIC (OSPF có xác thực)

```
interface Serial0/0/0
 ip address 192.168.50.74 255.255.255.252
 ip ospf message-digest-key 1 md5 SunCareOSPFKey2026
interface GigabitEthernet0/0
 ip address 192.168.51.1 255.255.255.240

router ospf 1
 router-id 2.2.2.2
 area 0 authentication message-digest
 network 192.168.51.0 0.0.0.15 area 0
 network 192.168.50.72 0.0.0.3 area 0

ntp server 192.168.50.73
logging host 192.168.50.69
```

## 3a. SW-DIST-SC (VLAN, Trunk, EtherChannel, STP)

```
vlan 10
 name NGOAITRU
vlan 20
 name NOITRU
vlan 30
 name IT_SERVER
vlan 40
 name GUEST
vlan 999
 name NATIVE_UNUSED

! EtherChannel LACP xuong SW-ACC-NGOAITRU (2 day; cong fa0/1-2 theo quy uoc Lab 01)
interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,999

! Trunk xuong SW-ACC-NOITRU (vi du cong fa0/3) - native 999
interface fa0/3
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,999

! Cac trunk toi R-HQ-1/R-HQ-2/WLC-SC/ASA-SC dung cung native 999 va allowed vlan tuong ung
spanning-tree vlan 10,20,30,40 priority 4096
```

## 4. SW-ACC-NGOAITRU / SW-ACC-NOITRU (Layer 2)

> **Lưu ý cập nhật:** bảng địa chỉ chưa cấp IP quản trị cho các switch. Để TACACS+/syslog/NTP chạy được từ switch cần thêm 1 SVI quản trị (ví dụ trong VLAN 30) `interface vlan 30` / `ip address <IP còn trống trong 192.168.50.64/29> 255.255.255.248` và `ip default-gateway 192.168.50.65`.

```
! SW-ACC-NGOAITRU
! Uplink EtherChannel - phai TRUST de DHCP Snooping/DAI khong chan DHCP offer tu R-HQ-1 va ARP tu gateway
interface range fa0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,999
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
! VLAN 40 (GUEST) chi di qua Wifi/WLC, khong co cong co day tren switch nay nen khong dua vao snooping/DAI
ip dhcp snooping vlan 10
ip arp inspection vlan 10

! SW-ACC-NOITRU
! Luu y: neu uplink len SW-DIST-SC nam trong fa0/1-15 thi loai cong do khoi range access va dat la trunk (native 999)
interface range fa0/1-15
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

## 5. WLC-SC (2 SSID)

```
config interface address management 192.168.50.68 255.255.255.248 192.168.50.65

config interface create NGOAITRU-DATA 10
config interface address dynamic-interface NGOAITRU-DATA 192.168.50.4 255.255.255.224 192.168.50.1
config interface vlan dynamic-interface NGOAITRU-DATA 10

config interface create GUEST-DATA 40
config interface address dynamic-interface GUEST-DATA 192.168.50.100 255.255.255.224 192.168.50.97
config interface vlan dynamic-interface GUEST-DATA 40

config wlan create 1 SUNCARE-STAFF SUNCARE-STAFF
config wlan interface 1 NGOAITRU-DATA
config wlan security wpa wpa2 enable 1
config wlan security wpa akm psk enable 1
config wlan security wpa akm psk set-key ascii Staff@SC2026 1
config wlan enable 1

config wlan create 2 SUNCARE-GUEST SUNCARE-GUEST
config wlan interface 2 GUEST-DATA
config wlan security wpa wpa2 enable 2
config wlan security wpa akm psk enable 2
config wlan security wpa akm psk set-key ascii Guest@SC2026 2
config wlan enable 2
```

## 6. ASA-SC (DMZ, chỉ mở 443)

> **Lưu ý cập nhật:** (1) Từ ASA 8.3 trở lên, ACL chiều `outside` phải tham chiếu **địa chỉ thật** của server (`192.168.61.10`), không dùng IP mapped (`198.18.10.10`). (2) IP mapped `.10` nằm ngoài subnet `/30` của link ASA–ISP-C (`198.18.10.0/30`): ISP-C cần có route `198.18.10.10/32` qua `198.18.10.1` (ASA outside) hoặc mở rộng link lên /28. (3) Dải `198.18.0.0/15` là không gian benchmark RFC 2544, không phải IP public thật — chỉ dùng cho lab, triển khai thật phải thay bằng IP public do ISP cấp. (4) ASA cũng cần `route inside` về mạng nội bộ qua R-HQ-1 (`192.168.50.65`) và các interface phải `no shutdown`; các dòng này chưa có trong cấu hình mẫu bên dưới.

```
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 198.18.10.1 255.255.255.252
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.50.66 255.255.255.248
interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 192.168.61.1 255.255.255.0

object network SRV-PORTAL-OBJ
 host 192.168.61.10
 nat (dmz,outside) static 198.18.10.10

! ASA 8.3+: ACL ngoai phai tham chieu IP THAT cua server (192.168.61.10), khong dung IP mapped 198.18.10.10
access-list OUTSIDE_IN extended permit tcp any host 192.168.61.10 eq 443
access-group OUTSIDE_IN in interface outside
route outside 0.0.0.0 0.0.0.0 198.18.10.2
```

---

## 6a. Automation (Task 20)

```python
# backup_devices.py - Netmiko backup show running-config (chay dinh ky bang cron/Task Scheduler de luu nhat ky tuan thu)
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
# deploy_clinic_2.yml - chi nham nhom clinic_2, khong dung HQ/Clinic dang chay (IP lay tu host_vars)
- hosts: clinic_2
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

## 7. Output verify tổng hợp

| Task | Lệnh | Kết quả mong đợi |
|---|---|---|
| 7 | `show ip ospf neighbor` (R-CLINIC) | `FULL` — nếu sai key/loại xác thực MD5, neighbor không bao giờ lên (hello bị drop; `debug ip ospf adj` báo "Mismatch Authentication"); kẹt vòng lặp `EXSTART` là dấu hiệu MTU mismatch |
| 10 | `ping 8.8.8.8` từ NOITRU và IT_SERVER | Cả 2 đều timeout |
| 11 | `ping` GUEST → NGOAITRU | Timeout |
| 13 | `show glbp brief` | 2 router cùng ở trạng thái forward (Active + Active, khác hẳn HSRP/VRRP chỉ 1 Active) |
| 19 | ASA: `telnet 198.18.10.10 80` từ ngoài | Timeout (ASA drop im lặng gói không khớp ACL, không gửi RST/ICMP; chỉ mở 443) |

## 8. Giải thích các điểm quan trọng / dễ sai

- **OSPF xác thực MD5 phải khớp key ở cả 2 đầu và bật đúng chế độ** (`area 0 authentication message-digest` + `ip ospf message-digest-key` trên interface) — thiếu 1 trong 2 dòng (hoặc sai key) khiến neighbor không bao giờ hình thành (hello bị drop, `debug ip ospf adj` báo "Mismatch Authentication"), dễ nhầm tưởng là lỗi Layer 1/2. Lưu ý: neighbor kẹt vòng lặp `EXSTART` là triệu chứng của MTU mismatch, không phải lỗi xác thực.
- **GLBP khác HSRP/VRRP ở chỗ cả 2 router cùng chuyển tiếp traffic** (Active Virtual Forwarder khác nhau cho từng client dựa trên round-robin MAC ảo) — không phải kiểu "1 router làm việc, 1 router chờ" như HSRP/VRRP; đây là lý do GLBP phù hợp cho VLAN NOITRU cần tận dụng cả 2 đường thay vì lãng phí 1 router rảnh rỗi.
- **OSPF không quảng bá NOITRU/IT_SERVER sang Clinic** chỉ khiến Clinic không có route cụ thể tới 2 vùng này, **chưa phải là cách ly**: Clinic vẫn có default route về R-HQ-1 và nếu R-HQ-1 không có ACL trên `Serial0/1/0` thì gói từ Clinic tới NOITRU/IT_SERVER vẫn được định tuyến tới đích. Cách ly thật cần ACL `CLINIC_IN` trên R-HQ-1 (xem mục 1).
- **ASA chỉ mở cổng 443, không mở 80** — với dữ liệu y tế, đây không phải tùy chọn mà gần như bắt buộc theo thông lệ bảo mật ngành (tránh truyền dữ liệu không mã hóa); học viên hay quên và mở luôn cả 80 "cho chắc" — đây chính là lỗi cần tránh.
- **2 SSID trên 1 WLC bắt buộc 2 dynamic interface riêng** (không dùng chung 1 interface cho 2 WLAN) — mỗi SSID phải ánh xạ đúng VLAN của mình, dùng chung interface sẽ khiến khách và nhân viên vô tình cùng chung 1 mạng.
