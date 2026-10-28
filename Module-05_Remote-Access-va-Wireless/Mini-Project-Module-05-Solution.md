# Mini Lab — Module 05: Remote Access và Wireless (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-05.md`

---

## 1. HSRP Plan hoàn chỉnh

| VLAN | R-HQ-1 (Active) | R-HQ-2 (Standby) | Virtual IP (gateway mới) |
|---|---|---|---|
| 10 (Sales) | 192.168.10.1 | 192.168.10.2 | **192.168.10.3** |
| 40 (IT/Server) | 192.168.10.65 | 192.168.10.77 | **192.168.10.78** |

## 2. WAN Failover Plan

| Link | Vai trò | Cơ chế phát hiện lỗi |
|---|---|---|
| R-HQ-1 ↔ ISP-A (`203.0.113.0/30`) | Chính | IP SLA ICMP echo tới `203.0.113.2`, gắn `track` object vào static default route |
| R-HQ-1 ↔ ISP-B (`198.51.100.0/30`, thêm mới) | Dự phòng | Floating static default route, AD cao hơn — tự nhận khi route chính bị track rớt |

## 3. WLAN Plan hoàn chỉnh

| SSID | VLAN | Bảo mật | Đối tượng |
|---|---|---|---|
| TECHSTART-CORP | 10 | WPA2-PSK (khóa riêng, đổi định kỳ) | Nhân viên |
| TECHSTART-GUEST | 20 | WPA2-PSK (khóa khác hoàn toàn) | Khách |

## 4. Cấu hình mẫu

### HSRP — R-HQ-1
```
interface GigabitEthernet0/0.10
 standby 10 ip 192.168.10.3
 standby 10 priority 110
 standby 10 preempt

interface GigabitEthernet0/0.40
 standby 40 ip 192.168.10.78
 standby 40 priority 110
 standby 40 preempt
```

### HSRP — R-HQ-2
```
interface GigabitEthernet0/0.10
 standby 10 ip 192.168.10.3
 standby 10 priority 100

interface GigabitEthernet0/0.40
 standby 40 ip 192.168.10.78
 standby 40 priority 100
```

### Rà soát lại DHCP (Module 04) — R-HQ-1
```
ip dhcp pool VLAN10_SALES
 default-router 192.168.10.3

! WLC dung 192.168.10.51 cho GUEST-DATA nen phai loai tru them .51 (Module 04 moi chi loai tru .49-.50)
ip dhcp excluded-address 192.168.10.51
```

### WAN Failover — R-HQ-1
```
interface Serial0/0/2
 ip address 198.51.100.1 255.255.255.252
 ip nat outside
 no shutdown

ip sla 1
 icmp-echo 203.0.113.2 source-interface Serial0/0/1
 frequency 5
ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability

ip route 0.0.0.0 0.0.0.0 203.0.113.2 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.2 200

! PAT theo tung ISP: moi duong ra dung dia chi cua chinh interface do
route-map NAT_ISPA permit 10
 match ip address NAT_PAT
 match interface Serial0/0/1
route-map NAT_ISPB permit 10
 match ip address NAT_PAT
 match interface Serial0/0/2
no ip nat inside source list NAT_PAT interface Serial0/0/1 overload
ip nat inside source route-map NAT_ISPA interface Serial0/0/1 overload
ip nat inside source route-map NAT_ISPB interface Serial0/0/2 overload
```

### VPN Remote Access (Easy VPN Server) — R-HQ-1
```
aaa new-model
aaa authentication login VPN_XAUTH local
aaa authorization network VPN_GROUP local
username remote01 secret Cisco@12345

ip local pool VPN_POOL 192.168.50.10 192.168.50.20

access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.50.0 0.0.0.255

! NAT exemption: traffic VPN toi pool 192.168.50.x khong duoc bi PAT (ACL NAT_PAT tu Module 04)
ip access-list extended NAT_PAT
 5 deny ip 192.168.10.0 0.0.0.255 192.168.50.0 0.0.0.255

crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2

crypto isakmp client configuration group TECHSTART-VPN
 key VpnPreShare2026
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
```
Remote PC dùng Cisco VPN Client: Server = `203.0.113.1`, Group = `TECHSTART-VPN` / key `VpnPreShare2026`, đăng nhập XAuth bằng `remote01`.

> **Lưu ý cập nhật:** Cisco VPN Client đã End-of-Life (khoảng 2011–2012) nên chỉ dùng trong lab; môi trường thực tế dùng Cisco Secure Client (AnyConnect) hoặc FlexVPN. Ngoài ra DH group 2 và SHA-1 đã yếu — nên ưu tiên `group 14` trở lên và `hash sha256`.

### WLC + WLAN (CLI tương đương trên WLC)
```
! Management interface cua WLC
config interface address management 192.168.10.68 255.255.255.240 192.168.10.78

! Dynamic interface anh xa VLAN
config interface create SALES-DATA 10
config interface address dynamic-interface SALES-DATA 192.168.10.4 255.255.255.224 192.168.10.3
config interface vlan SALES-DATA 10

config interface create GUEST-DATA 20
config interface address dynamic-interface GUEST-DATA 192.168.10.51 255.255.255.240 192.168.10.49
config interface vlan GUEST-DATA 20

! WLAN CORP
config wlan create 1 TECHSTART-CORP TECHSTART-CORP
config wlan interface 1 SALES-DATA
config wlan security wpa enable 1
config wlan security wpa wpa2 enable 1
config wlan security wpa akm 802.1x disable 1
config wlan security wpa akm psk enable 1
config wlan security wpa akm psk set-key ascii Corp@Secure2026 1
config wlan enable 1

! WLAN GUEST
config wlan create 2 TECHSTART-GUEST TECHSTART-GUEST
config wlan interface 2 GUEST-DATA
config wlan security wpa enable 2
config wlan security wpa wpa2 enable 2
config wlan security wpa akm 802.1x disable 2
config wlan security wpa akm psk enable 2
config wlan security wpa akm psk set-key ascii Guest@Open2026 2
config wlan enable 2
```

Switch nối WLC: cấu hình **trunk**, allowed VLAN 10,20,40,999 (WLC tự tag traffic client theo dynamic interface). Switch nối từng AP: cấu hình **access port VLAN 40** (AP chỉ cần VLAN quản lý để CAPWAP tunnel về WLC, không cần trunk xuống AP).

> **Lưu ý cập nhật:** AP nối SW-ACC2 nằm trong VLAN 40 nên trunk SW-ACC1–SW-ACC2 phải được thêm VLAN 40 vào danh sách allowed (hiện chỉ cho phép 10,30,999), và VLAN 40 phải có DHCP scope kèm option 43 trỏ về WLC (`192.168.10.68`) để AP tìm được controller. Nếu không muốn sửa trunk/DHCP, có thể đặt AP vào VLAN 10 thay vì VLAN 40.

## 5. Output verify mẫu

```
R-HQ-1# show standby brief
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/0.10    10   110 P Active  local           192.168.10.2    192.168.10.3
```

```
R-HQ-1# show ip route static
S*   0.0.0.0/0 [1/0] via 203.0.113.2
```
Khi tắt thử `Se0/0/1`: route trên biến mất khỏi bảng, route AD 200 qua `198.51.100.2` thay thế ngay.

> **Lưu ý cập nhật:** `show ip route` không hiển thị chữ `track 1`; kiểm tra trạng thái track bằng `show track 1` và `show ip sla statistics`. Ngoài ra, tắt `Se0/0/1` chỉ kiểm tra được trường hợp interface down; để kiểm tra đúng cơ chế IP SLA (link vẫn up nhưng ISP-A không phản hồi), hãy chặn ICMP tới `203.0.113.2` tại phía ISP rồi quan sát track chuyển Down.

## 6. Giải thích các điểm quan trọng / dễ sai

- **Cả 2 router HSRP phải cùng 1 virtual IP, khác priority** — nếu quên đặt priority, kết quả Active/Standby phụ thuộc vào IP vật lý (router IP cao hơn thắng), không kiểm soát được ai là Active theo ý muốn.
- **`preempt` bắt buộc nếu muốn R-HQ-1 tự giành lại vai trò Active sau khi hồi phục** — thiếu dòng này, R-HQ-2 vẫn giữ Active mãi dù R-HQ-1 đã online trở lại, gây lệch tải không mong muốn.
- **Floating static route không tự động "thông minh"** — nếu không gắn `track`, route dự phòng dù có AD cao hơn vẫn không bao giờ được dùng cho tới khi route chính bị xóa khỏi bảng định tuyến theo cách nào đó; `track` chính là cơ chế xóa route chính khi ISP-A mất kết nối.
- **VPN Remote Access chỉ cho vào được mạng trực tiếp nối R-HQ-1** (`192.168.10.0/24` qua ACL 101) — muốn remote user vào cả Branch/DR cần mở rộng ACL 101 và cân nhắc redistribute route pool VPN vào OSPF, ngoài phạm vi yêu cầu bài này.
- **Switchport nối AP là access port, không phải trunk** — sinh viên hay nhầm rằng AP cần trunk để "chạy nhiều VLAN"; thực tế kiến trúc lightweight AP đẩy việc phân biệt VLAN client về phía WLC (qua CAPWAP), AP chỉ cần 1 VLAN quản lý duy nhất.
