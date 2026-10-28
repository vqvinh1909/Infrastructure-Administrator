# Mini Lab — Module 04: Dịch vụ và bảo mật cơ bản (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-04.md`

---

## 1. DHCP Pool Plan hoàn chỉnh

| VLAN | Dải cấp DHCP | Loại trừ (exclude) | Cấp bởi |
|---|---|---|---|
| VLAN 10 (Sales) | 192.168.10.1 – .30 | .1 – .5 (gateway + dự phòng thiết bị tĩnh) | R-HQ (pool cục bộ) |
| VLAN 20 (Guest) | 192.168.10.49 – .62 | .49 – .50 (gateway + dự phòng) | R-HQ (pool cục bộ) |
| VLAN 30 (Kế toán/NS) | 192.168.10.34 – .46 | .33 (gateway) | SRV-DHCP (192.168.10.67), qua relay |

## 2. ACL Plan hoàn chỉnh

| Vị trí | Loại ACL | Mục đích |
|---|---|---|
| `Gi0/0.20` (Guest) chiều `in` | Extended | Chặn Guest tới `192.168.10.0/24` (3 dải VLAN còn lại), cho phép mọi thứ khác (Internet) |
| `Se0/2/0` (hướng Partner) chiều `in` | Extended | Cho phép EIGRP (giữ neighbor sống) + TCP 80/443 tới địa chỉ public của SRV-WEB, chặn phần còn lại |

## 3. Cấu hình mẫu — R-HQ

### DHCP
```
ip dhcp excluded-address 192.168.10.1 192.168.10.5
ip dhcp excluded-address 192.168.10.49 192.168.10.50

ip dhcp pool VLAN10_SALES
 network 192.168.10.0 255.255.255.224
 default-router 192.168.10.1
 dns-server 8.8.8.8

ip dhcp pool VLAN20_GUEST
 network 192.168.10.48 255.255.255.240
 default-router 192.168.10.49
 dns-server 8.8.8.8

interface GigabitEthernet0/0.30
 ip helper-address 192.168.10.67
```
SRV-DHCP (server riêng trong VLAN 40) cấu hình pool cho `192.168.10.32/28`, default gateway `.33`, loại trừ `.33`, qua giao diện GUI của server.

### ACL Guest (chặn nội bộ, cho ra Internet)
```
ip access-list extended GUEST_ISOLATION
 deny ip 192.168.10.48 0.0.0.15 192.168.10.0 0.0.0.31
 deny ip 192.168.10.48 0.0.0.15 192.168.10.32 0.0.0.15
 deny ip 192.168.10.48 0.0.0.15 192.168.10.64 0.0.0.15
 deny ip 192.168.10.48 0.0.0.15 192.168.20.0 0.0.0.255
 deny ip 192.168.10.48 0.0.0.15 192.168.30.0 0.0.0.255
 deny ip 192.168.10.48 0.0.0.15 192.168.40.0 0.0.0.255
 deny ip 192.168.10.48 0.0.0.15 172.16.0.0 0.0.255.255
 permit ip any any

interface GigabitEthernet0/0.20
 ip access-group GUEST_ISOLATION in
```

### NAT
```
! ACL NAT dang extended de loai tru traffic HQ -> Partner (Se0/2/0 la outside nen neu khong deny se bi PAT nham)
ip access-list extended NAT_PAT
 deny ip any 192.168.40.0 0.0.0.255
 deny ip any 172.16.1.8 0.0.0.3
 permit ip 192.168.10.0 0.0.0.255 any
 permit ip 192.168.20.0 0.0.0.255 any
 permit ip 192.168.30.0 0.0.0.255 any

interface Serial0/0/1
 ip nat outside
interface Serial0/2/0
 ip nat outside
interface GigabitEthernet0/0.10
 ip nat inside
interface GigabitEthernet0/0.20
 ip nat inside
interface GigabitEthernet0/0.30
 ip nat inside
interface GigabitEthernet0/0.40
 ip nat inside
interface Serial0/1/0
 ip nat inside
interface Serial0/1/1
 ip nat inside

ip nat inside source list NAT_PAT interface Serial0/0/1 overload
ip nat inside source static tcp 192.168.10.70 80 203.0.113.10 80 extendable
ip nat inside source static tcp 192.168.10.70 443 203.0.113.10 443 extendable
```

### ACL Partner (chỉ cho phép EIGRP + web đúng port)
```
ip access-list extended PARTNER_IN
 permit eigrp host 172.16.1.10 host 224.0.0.10
 permit eigrp host 172.16.1.10 host 172.16.1.9
 permit udp host 172.16.1.10 host 172.16.1.9 eq ntp
 permit udp host 172.16.1.10 host 192.168.10.66 eq syslog
 permit tcp 192.168.40.0 0.0.0.255 host 203.0.113.10 eq 80
 permit tcp 192.168.40.0 0.0.0.255 host 203.0.113.10 eq 443
 deny ip any any log

interface Serial0/2/0
 ip access-group PARTNER_IN in
```

### NTP & Syslog
```
ntp master 3
service timestamps log datetime msec

logging host 192.168.10.66
logging trap informational
```

> **Lưu ý cập nhật:** ACL `PARTNER_IN` kết thúc bằng `deny ip any any log` nên nếu thiếu 2 dòng `permit udp ... eq ntp` / `eq syslog` ở trên thì NTP và syslog của R-PARTNER bị chặn. Để syslog tới `192.168.10.66` thực sự đi được, R-PARTNER còn cần route tới dải `192.168.10.0/24` (hiện chỉ có route tới `203.0.113.10`); nếu không muốn mở thêm, có thể loại R-PARTNER khỏi NTP/syslog và bỏ 2 dòng permit này.

## 4. Cấu hình mẫu — R-BRANCH / R-DR (NTP client)
```
ntp server 172.16.1.1
service timestamps log datetime msec
logging host 192.168.10.66
```

## 5. Cấu hình mẫu — R-PARTNER

```
ntp server 172.16.1.9
service timestamps log datetime msec
logging host 192.168.10.66

! Partner khong tham gia OSPF noi bo nen khong biet duong toi dia chi
! public 203.0.113.10 cua SRV-WEB - can 1 static route rieng toi R-HQ:
ip route 203.0.113.10 255.255.255.255 172.16.1.9
```

## 6. Output verify mẫu

```
R-HQ# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.10:80    192.168.10.70:80   192.168.40.10:1500 192.168.40.10:1500
```

```
R-HQ# show access-lists PARTNER_IN
Extended IP access list PARTNER_IN
    10 permit eigrp host 172.16.1.10 host 224.0.0.10 (28 matches)
    30 permit tcp 192.168.40.0 0.0.0.255 host 203.0.113.10 eq www (12 matches)
    50 deny ip any any log (3 matches)
```

## 7. Giải thích các điểm quan trọng / dễ sai

- **ACL trên `Se0/2/0` phải chừa đường cho EIGRP hello (multicast 224.0.0.10)** — đây là lỗi rất hay gặp: học viên viết ACL chặn Partner xong thì neighbor EIGRP rớt ngay lập tức vì hello packet cũng bị deny theo luật cuối `deny ip any any`.
- **ACL kiểm tra ở địa chỉ nào phụ thuộc hướng NAT** — ACL `in` trên interface `outside` (`Se0/2/0`) nhìn thấy địa chỉ đích là **địa chỉ public đã NAT** (`203.0.113.10`), không phải địa chỉ nội bộ thật (`192.168.10.70`) — viết ACL theo IP nội bộ ở bước này sẽ không bao giờ match.
- **R-PARTNER cần 1 static route riêng tới `203.0.113.10`** vì địa chỉ này không tồn tại trong OSPF domain và Partner không chạy OSPF — quên bước này thì dù NAT/ACL đúng 100%, Partner vẫn không gửi được gói tin tới nơi.
- **DHCP Relay chỉ cần đặt `ip helper-address` phía router (subinterface VLAN 30)**, không cấu hình gì thêm phía SRV-DHCP ngoài pool bình thường — nhầm lẫn phổ biến là đặt `ip helper-address` trên SRV-DHCP hoặc trên cả 2 phía.
- **`ntp master` chỉ nên bật ở đúng 1 điểm** (R-HQ) — nếu bật `ntp master` ở nhiều router cùng lúc, mỗi router tự cho mình là nguồn giờ chuẩn, dễ gây xung đột đồng bộ.
