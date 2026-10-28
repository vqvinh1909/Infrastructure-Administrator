# Mini Lab — Module 06: Bảo mật nâng cao (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-06.md`

---

## 1. Security Level Plan hoàn chỉnh (ASA)

| Interface | Vùng (nameif) | Security level | Lý do |
|---|---|---|---|
| Gi0/0 (nối ISP-A) | outside | 0 | Vùng không tin cậy nhất — Internet |
| Gi0/1 (nối R-HQ-1) | inside | 100 | Vùng tin cậy tuyệt đối — toàn bộ mạng nội bộ công ty |
| Gi0/2 (nối SRV-WEB) | dmz | 50 | Tin cậy trung bình — public dịch vụ ra ngoài nhưng không được toàn quyền vào inside |

## 2. Layer 2 Hardening Plan

| Cơ chế | Áp dụng | Mục đích |
|---|---|---|
| Port Security | Toàn bộ access port SW-ACC1/ACC2 (nối máy người dùng) | Giới hạn 2 MAC/port, chặn cắm switch lạ mở rộng port trái phép |
| DHCP Snooping | Trust trên port uplink (nối SW-DIST/switch khác); untrust (mặc định) trên access port | Chặn DHCP server giả mạo cắm vào port nhân viên |
| Dynamic ARP Inspection | VLAN người dùng (10, 20, 30), dựa trên DHCP snooping binding | Chặn ARP giả mạo/man-in-the-middle trong VLAN |

## 3. Cấu hình mẫu — Layer 2 (SW-ACC1)

```
interface range fa0/5-9, fa0/20-24
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

ip dhcp snooping
ip dhcp snooping vlan 10,20

interface port-channel 1
 ip dhcp snooping trust
 ip arp inspection trust
interface fa0/3
 ip dhcp snooping trust
 ip arp inspection trust

ip arp inspection vlan 10,20
```

### SW-ACC2 (tương tự, VLAN 30)
```
interface range fa0/5-15
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

ip dhcp snooping
ip dhcp snooping vlan 30
interface fa0/4
 ip dhcp snooping trust
 ip arp inspection trust

ip arp inspection vlan 30
```

## 4. Cấu hình mẫu — AAA (áp dụng mẫu trên mọi router/switch)

```
tacacs server SRV-AAA
 address ipv4 192.168.10.69
 key TacacsKey2026

aaa new-model
aaa authentication login VTY_AUTH group tacacs+ local

username backupadmin secret Local@Backup2026

line vty 0 4
 login authentication VTY_AUTH
line console 0
 login authentication VTY_AUTH
```

## 5. Cấu hình mẫu — Cisco ASA

> **Lưu ý cập nhật:** ASA 5500-X/5506-X đã End-of-Sale (5506-X hết hỗ trợ ngày 2026-08-31); dòng hiện nay là Cisco Secure Firewall. ASA trong bài này chỉ dùng cho lab/hệ thống legacy.

```
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.99.1 255.255.255.252
 no shutdown

interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 192.168.100.1 255.255.255.0
 no shutdown

object network INSIDE-NET
 subnet 192.168.0.0 255.255.0.0
 nat (inside,outside) dynamic interface

object network SRV-WEB-OBJ
 host 192.168.100.10
 nat (dmz,outside) static 203.0.113.10

access-list OUTSIDE_IN extended permit tcp any host 192.168.100.10 eq www
access-list OUTSIDE_IN extended permit tcp any host 192.168.100.10 eq https
access-group OUTSIDE_IN in interface outside

route outside 0.0.0.0 0.0.0.0 203.0.113.2
route inside 192.168.0.0 255.255.0.0 192.168.99.2
route inside 172.16.0.0 255.255.0.0 192.168.99.2
```
Traffic từ `inside` (R-HQ-1) sang `dmz` (SRV-WEB) không cần ACL riêng — security level 100 → 50 được phép mặc định trên ASA (chỉ chiều cao xuống thấp).

> **Lưu ý cập nhật:** từ ASA 8.3 trở lên, ACL trên interface `outside` được so khớp với địa chỉ **thật** (`192.168.100.10`) vì ASA un-NAT gói tin trước khi kiểm tra ACL — nên không dùng IP public `203.0.113.10` trong `OUTSIDE_IN`. Ngoài ra ASA cần route `inside` về các mạng nội bộ (`192.168.0.0/16`, `172.16.0.0/16`) qua R-HQ-1 (`192.168.99.2`), nếu không chỉ có default route ra `outside` thì traffic trả về mạng nội bộ không biết đi đâu.

## 6. Cấu hình mẫu — dọn dẹp & cập nhật R-HQ-1

```
! Go interface cu noi thang ISP-A, thay bang noi vao ASA inside
interface Serial0/0/1
 no ip nat outside
 no crypto map VPN_MAP
 ip address 192.168.99.2 255.255.255.252

! Xoa route/track cu toi ISP-A truc tiep (Module 05), thay bang default qua ASA
no ip route 0.0.0.0 0.0.0.0 203.0.113.2 track 1
no ip route 0.0.0.0 0.0.0.0 198.51.100.2 200
no track 1
no ip sla schedule 1
no ip sla 1
ip route 0.0.0.0 0.0.0.0 192.168.99.1

! Them route toi DMZ vi SRV-WEB da chuyen khoi VLAN 40
ip route 192.168.100.0 255.255.255.0 192.168.99.1

! Xoa static NAT cu cho SRV-WEB (Module 04) - ASA da dam nhan
no ip nat inside source static tcp 192.168.10.70 80 203.0.113.10 80 extendable
no ip nat inside source static tcp 192.168.10.70 443 203.0.113.10 443 extendable

! Xoa PAT overload cu (Module 04/05) - ASA dam nhan NAT ra Internet
no ip nat inside source route-map NAT_ISPA interface Serial0/0/1 overload
no ip nat inside source route-map NAT_ISPB interface Serial0/0/2 overload
no ip nat inside source list NAT_PAT interface Serial0/0/1 overload
```

### Cập nhật ACL Partner (đích giờ là DMZ thật, không phải IP NAT public nữa)
```
ip access-list extended PARTNER_IN
 permit eigrp host 172.16.1.10 host 224.0.0.10
 permit eigrp host 172.16.1.10 host 172.16.1.9
 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 80
 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 443
 deny ip any any log
```

### Cập nhật R-PARTNER (route tới SRV-WEB theo IP thật trong DMZ)
```
no ip route 203.0.113.10 255.255.255.255 172.16.1.9
ip route 192.168.100.10 255.255.255.255 172.16.1.9
```

> **Lưu ý cập nhật:** sau khi thêm ASA, ACL `PARTNER_IN` đã đổi đích sang `192.168.100.10` nên R-PARTNER cũng phải đổi route tương ứng (có thể thay bằng quảng bá qua EIGRP). Các lệnh dọn dẹp ở mục 6 gỡ toàn bộ NAT overload, IP SLA/track, floating route và crypto map cũ trên R-HQ-1 (nếu muốn giữ VPN remote access thì phải chuyển điểm kết thúc VPN sang ASA, ngoài phạm vi bài này); các lệnh `ip nat inside`/`ip nat outside` còn sót trên các interface còn lại cũng nên gỡ.

## 7. Output verify mẫu

```
SW-ACC1# show port-security interface fa0/5
Port Security              : Enabled
Violation Mode             : Restrict
Maximum MAC Addresses      : 2
Sticky MAC Addresses       : 1
```

```
ASA# show nat
Auto NAT Policies (Section 2)
1 (dmz) to (outside) source static SRV-WEB-OBJ 203.0.113.10
    translate_hits = 0, untranslate_hits = 0
```

## 8. Giải thích các điểm quan trọng / dễ sai

- **Thứ tự `aaa authentication login ... group tacacs+ local`** rất quan trọng — nếu chỉ ghi `group tacacs+` mà không có `local` phía sau, khi SRV-AAA sập, toàn bộ đội IT bị khóa khỏi thiết bị hoàn toàn, kể cả console.
- **`ip arp inspection` chỉ có tác dụng nếu đã bật `ip dhcp snooping` trước** — DAI dùng chính bảng binding của DHCP snooping để biết MAC/IP nào hợp lệ; bật DAI mà chưa có snooping thì mọi ARP hoặc bị chặn hết hoặc không kiểm tra được gì tùy cấu hình.
- **Sau khi chạy các lệnh dọn dẹp ở mục 6, R-HQ-1 không còn `ip nat outside`/`ip nat inside` hay lệnh PAT overload liên quan tới Internet nữa** — vai trò NAT ra Internet đã chuyển hẳn cho ASA; nếu quên xóa cấu hình NAT cũ, dễ gây xung đột hoặc route loop khi debug.
- **DMZ không tự động thấy được bởi inside nếu thiếu route cụ thể** — R-HQ-1 chỉ có default route qua ASA thì vẫn đi được (default route bắt được luôn), nhưng thêm route cụ thể `192.168.100.0/24` giúp rõ ràng và dễ audit hơn, tránh phụ thuộc hoàn toàn vào 1 dòng default.
- **Security level là nguyên tắc mặc định, không thay thế ACL khi cần lọc chi tiết** — chiều cao→thấp (inside→dmz) được phép mặc định, nhưng chiều thấp→cao hoặc bất kỳ traffic nào từ `outside` luôn cần ACL tường minh, đây là lý do `OUTSIDE_IN` bắt buộc phải có dù đã có security level.
