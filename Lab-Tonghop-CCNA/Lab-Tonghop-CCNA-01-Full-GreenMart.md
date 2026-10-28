# Lab Tổng Hợp CCNA — Full: Hạ tầng mạng GreenMart Retail (Module 01–07)

> Kết hợp toàn bộ 7 module (Bài 01–30) trên **1 topology duy nhất** — đúng tinh thần "lab tổng hợp" CCNA: 1 sơ đồ, danh sách Task nối tiếp, mỗi task có kết quả kiểm tra cụ thể ngay trong đề (giống format đề thi/lab tổng hợp thực tế).
> File: `Lab-Tonghop-CCNA-01-Full-GreenMart.md` · Lời giải: `Lab-Tonghop-CCNA-01-Full-GreenMart-Solution.md`
> Môi trường: Cisco Packet Tracer + Python/Ansible.

---

## Bối cảnh

**GreenMart Retail** — chuỗi bán lẻ có 1 văn phòng chính (HQ), 2 cửa hàng (Store A, Store B), 1 website bán hàng online cần public ra Internet. Bạn là kỹ sư mạng duy nhất, dựng toàn bộ hạ tầng từ Lớp 2 tới automation trong 1 lần triển khai.

## Sơ đồ mạng (đề bài)

```
Internet: [ISP-A] (duong chinh) va [ISP-B] (du phong) noi vao R-HQ-1
          [ISP-C] (duong rieng) noi vao ASA-GM cho vung DMZ

[ R-HQ-1 ]  (HSRP Active VLAN OFFICE/IT_SERVER, OSPF, NAT, VPN, WAN failover)
      |
[ SW-DIST-GM ]
      |
      |------ [ WLC-GM ] --- AP --- SSID GREENMART-SCANNER (VLAN WAREHOUSE)
      |
      || (EtherChannel LACP, 2 day du phong)
      |
[ SW-ACC-GM ]  (access port: VLAN OFFICE, VLAN WAREHOUSE)

[ R-HQ-2 ]  (HSRP Standby) cung noi vao SW-DIST-GM

[ ASA-GM ] --- inside --- noi vao SW-DIST-GM (VLAN IT_SERVER)
      |
     dmz
      |
[ SRV-WEB ]  (public qua ISP-C, NAT tinh)

[ R-HQ-1 ] --- Se0/1/0 (OSPF) --- [ R-STOREA ]  LAN Store A
[ R-HQ-1 ] --- Se0/1/1 (OSPF) --- [ R-STOREB ]  LAN Store B
```

## Bảng địa chỉ (cho trước — cấu hình đúng theo bảng, không tự đổi số)

| Đối tượng | Địa chỉ | Ghi chú |
|---|---|---|
| VLAN 10 — OFFICE | `10.10.0.0/27` | R-HQ-1: `.1`, R-HQ-2: `.2`, HSRP VIP: `.3` |
| VLAN 20 — WAREHOUSE | `10.10.0.32/27` | Gateway `.33` |
| VLAN 30 — IT_SERVER | `10.10.0.64/29` | R-HQ-1: `.65`; SRV-AAA `.67`; ASA inside `.66`; SRV-SYSLOG `.69`; WLC mgmt `.68` |
| WAN HQ↔StoreA | `10.10.0.72/30` | R-HQ-1 `.73`, R-STOREA `.74` |
| WAN HQ↔StoreB | `10.10.0.76/30` | R-HQ-1 `.77`, R-STOREB `.78` |
| WAN HQ↔ISP-A | `10.10.0.80/30` | R-HQ-1 `.81`, ISP-A `.82` |
| WAN HQ↔ISP-B (dự phòng) | `198.51.100.0/30` | R-HQ-1 `Se0/0/2` `.1`, ISP-B `.2` |
| LAN Store A | `10.10.1.0/28` | Gateway `.1` |
| LAN Store B | `10.10.2.0/28` | Gateway `.1` |
| VPN Pool (remote user) | `10.10.50.0/24` | |
| ASA outside (ISP-C) | `200.100.50.0/30` | ASA `.1`, ISP-C `.2` |
| DMZ (SRV-WEB) | `172.20.0.0/24` | SRV-WEB `.10` |
| IPv6 công ty | `2001:db8:aabb::/48` | Chia `/64` theo VLAN — tự hoàn thành ở Task 2 |

---

## Task 1 — Chọn phương tiện truyền dẫn

SW-DIST-GM đặt tại Server Room (tầng 3). SW-ACC-GM đặt tại tầng 1, cách 90m, đường đi gần khu vực máy nén điều hòa trung tâm (nhiễu điện mạnh). Chọn loại cáp cho đoạn này và giải thích lý do (liên hệ Bài 03).

**Kết quả cần đạt:** bảng chọn cáp + lý do kỹ thuật (suy hao/nhiễu điện từ/khoảng cách) — không có lệnh CLI để verify, chấm bằng lý luận đúng bản chất.

## Task 2 — IPv6

Từ khối `2001:db8:aabb::/48`, chia 1 subnet `/64` cho VLAN OFFICE và 1 subnet `/64` cho VLAN WAREHOUSE, gán trên đúng subinterface tương ứng ở R-HQ-1.

**Kết quả kiểm tra:**
```
R-HQ-1# show ipv6 interface brief
GigabitEthernet0/0.10  [up/up]  FE80::... 2001:DB8:AABB:1::1
GigabitEthernet0/0.20  [up/up]  FE80::... 2001:DB8:AABB:2::1
```

## Task 3 — VLAN & Trunk

Tạo VLAN 10 (OFFICE), 20 (WAREHOUSE), 30 (IT_SERVER) trên SW-DIST-GM và SW-ACC-GM. Trunk giữa 2 switch, đổi Native VLAN sang VLAN 999 (không gán host).

**Kết quả kiểm tra:**
```
SW-DIST-GM# show interfaces trunk
Port      Native vlan   Vlans allowed on trunk
Po1       999           10,20,30,999
```

## Task 4 — EtherChannel

Giữa SW-DIST-GM và SW-ACC-GM có 2 dây vật lý dự phòng — gộp thành EtherChannel LACP (Port-channel 1), không để STP khóa chết 1 dây.

**Kết quả kiểm tra:**
```
SW-DIST-GM# show etherchannel summary
1      Po1(SU)         LACP      Fa0/1(P) Fa0/2(P)
```

## Task 5 — STP Root Bridge

SW-DIST-GM phải luôn là Root Bridge cho VLAN 10, 20, 30.

**Kết quả kiểm tra:**
```
SW-DIST-GM# show spanning-tree vlan 10
Root ID    Priority 4106
           This bridge is the root
Bridge ID  Priority 4106  (priority 4096 sys-id-ext 10)
```

## Task 6 — Router-on-a-stick

R-HQ-1 định tuyến 3 VLAN qua subinterface trên `Gi0/0`, đúng địa chỉ ở bảng trên.

**Kết quả kiểm tra:**
```
R-HQ-1# show ip interface brief | include GigabitEthernet0/0
GigabitEthernet0/0.10  10.10.0.1     up   up
GigabitEthernet0/0.20  10.10.0.33    up   up
GigabitEthernet0/0.30  10.10.0.65    up   up
```

## Task 7 — OSPF Area 0

R-HQ-1, R-STOREA, R-STOREB chạy chung OSPF process 1, area 0, quảng bá đủ toàn bộ LAN/WAN liên quan.

**Kết quả kiểm tra:**
```
R-STOREA# show ip route ospf
O    10.10.0.0/27 [110/65] via 10.10.0.73
O    10.10.0.32/27 [110/65] via 10.10.0.73
O    10.10.2.0/28 [110/129] via 10.10.0.73
```

## Task 8 — Default route

R-HQ-1 có default route tới ISP-A (`10.10.0.82`), quảng bá lại vào OSPF để Store A/B tự học, không cấu hình tay từng router.

**Kết quả kiểm tra:**
```
R-STOREA# show ip route | include ^S\*|^O\*E2
O*E2  0.0.0.0/0 [110/1] via 10.10.0.73
```

## Task 9 — DHCP

VLAN OFFICE và WAREHOUSE nhận IP tự động từ R-HQ-1, loại trừ đúng dải gateway/HSRP.

**Kết quả kiểm tra:**
```
R-HQ-1# show ip dhcp binding
10.10.0.10   0100.1234.5678   ...
10.10.0.40   0100.aabb.ccdd   ...
```

## Task 10 — NAT

Toàn bộ mạng nội bộ (`10.10.0.0/16`) ra Internet qua NAT overload tại `Serial0/0/1`.

> **Lưu ý cập nhật:** cột Inside global là địa chỉ của `Serial0/0/1` (`10.10.0.81`) vì link ISP-A trong lab dùng dải private theo bảng địa chỉ; nếu thay bằng IP public thật cho link ISP-A thì Inside global sẽ là IP public đó.

**Kết quả kiểm tra:**
```
R-HQ-1# show ip nat translations
icmp 10.10.0.81:1  10.10.1.5:1  8.8.8.8:1  8.8.8.8:1
```

## Task 11 — ACL cách ly VLAN

VLAN WAREHOUSE không được truy cập VLAN OFFICE và ngược lại (2 chiều), cả 2 vẫn ra Internet bình thường.

**Kết quả kiểm tra:**
```
PC (WAREHOUSE)> ping 10.10.0.10   -> Reply from 10.10.0.33: Destination host unreachable (dung - ACL deny: router tra ICMP administratively prohibited)
PC (WAREHOUSE)> ping 8.8.8.8       -> Reply (dung)
```

## Task 12 — NTP & Syslog

R-HQ-1 làm NTP master, Store A/B đồng bộ theo; toàn bộ router gửi log về SRV-SYSLOG (`10.10.0.69`).

**Kết quả kiểm tra:**
```
R-STOREA# show ntp status
Clock is synchronized, stratum 4, reference is 10.10.0.73
```

## Task 13 — HSRP

VLAN OFFICE có 2 gateway (R-HQ-1 Active, R-HQ-2 Standby) dùng chung VIP `10.10.0.3`; tắt thử R-HQ-1 không được mất kết nối VLAN OFFICE.

**Kết quả kiểm tra:**
```
R-HQ-2# show standby brief
! Sau khi tat R-HQ-1 (R-HQ-2 can "standby 10 preempt" de co co P; Standby = unknown vi khong con router nao khac):
Gi0/0.10  10  100  P Active  local  unknown  10.10.0.3
```

## Task 14 — WAN Failover

R-HQ-1 có thêm link dự phòng ISP-B (`Se0/0/2`), dùng IP SLA + tracking để tự chuyển khi ISP-A mất, không cấu hình tay.

**Kết quả kiểm tra:**
```
! Tat Se0/0/1:
R-HQ-1# show ip route static
S*  0.0.0.0/0 [200/0] via 198.51.100.2
```

## Task 15 — VPN Remote Access

Nhân viên làm việc từ xa kết nối VPN vào R-HQ-1 (`Easy VPN Server`), sau khi kết nối truy cập được tài nguyên VLAN OFFICE.

**Kết quả kiểm tra:**
```
PC-remote (sau khi VPN)> ping 10.10.0.10   -> Reply
```

## Task 16 — Wireless / WLC

WLC-GM quản lý ít nhất 1 AP tại khu Kho vận, phát SSID `GREENMART-SCANNER` ánh xạ VLAN WAREHOUSE.

**Kết quả kiểm tra:**
```
WLC-GM# show wlan summary
WLAN ID  WLAN Profile Name / SSID     Status   Interface
1        GREENMART-SCANNER            Enabled  WAREHOUSE-DATA
```

## Task 17 — Layer 2 hardening

Toàn bộ access port trên SW-ACC-GM: Port Security (max 2 MAC, sticky, violation restrict), DHCP Snooping, Dynamic ARP Inspection.

**Kết quả kiểm tra:**
```
SW-ACC-GM# show port-security interface fa0/5
Violation Mode: Restrict   Maximum MAC: 2   Sticky: 1
```

## Task 18 — AAA

Đăng nhập quản trị mọi thiết bị qua TACACS+ (`SRV-AAA`, `10.10.0.67`), có tài khoản local dự phòng khi server lỗi.

**Kết quả kiểm tra:** tắt thử SRV-AAA, vẫn đăng nhập được vào switch bằng tài khoản local.

## Task 19 — ASA Firewall & DMZ

SRV-WEB (`172.20.0.10`) đặt sau ASA-GM, dùng đường ISP-C riêng. Internet chỉ vào được đúng cổng 80/443 qua NAT tĩnh `200.100.50.10`.

**Kết quả kiểm tra:**
```
ASA-GM# show nat
Auto NAT Policies (Section 2)
1 (dmz) to (outside) source static SRV-WEB-OBJ 200.100.50.10
    translate_hits = 0, untranslate_hits = 0
```
```
PC ngoai Internet> telnet 200.100.50.10 22   -> Timeout (dung - ASA drop im lang goi khong khop ACL, khong gui RST/ICMP; chi mo 80/443)
```

## Task 20 — Automation

Viết 1 script Python (Netmiko) backup `show running-config` toàn bộ thiết bị, và 1 Ansible playbook triển khai VLAN/IP khung cho Store C mới mở — không đụng cấu hình Store A/B đang chạy.

**Kết quả kiểm tra:** thư mục backup đủ file theo từng thiết bị; `ansible-playbook` chạy lần 2 báo `changed=0`.

---

## Ràng buộc chung

- Không dùng static route thay cho OSPF giữa các site (trừ đúng 1 chỗ: default route ra Internet ở Task 8).
- Không đổi bảng địa chỉ đã cho ở trên.
- Thứ tự làm Task nên theo đúng 1→20 vì các Task sau phụ thuộc kết quả Task trước (ví dụ Task 13 cần Task 9 đã ổn định).

## Tiêu chí đánh giá (Rubric)

| Nhóm Task | Trọng số |
|---|---|
| Task 1–2 (Nền tảng) | 5% |
| Task 3–5 (Switching) | 15% |
| Task 6–8 (Routing) | 15% |
| Task 9–12 (Dịch vụ cơ bản) | 15% |
| Task 13–16 (Remote/Wireless) | 20% |
| Task 17–19 (Bảo mật nâng cao) | 20% |
| Task 20 (Automation) | 10% |

Mỗi Task chấm đạt/không đạt dựa đúng vào khối "Kết quả kiểm tra" — không đạt nếu output thực tế khác với kỳ vọng, kể cả khi cấu hình "trông có vẻ đúng".

---

**Lưu ý khi viết:** không chèn cấu hình/lệnh sửa lỗi hoàn chỉnh vào file đề bài — lời giải mẫu đầy đủ (cấu hình từng thiết bị) để riêng ở `Lab-Tonghop-CCNA-01-Full-GreenMart-Solution.md`.
