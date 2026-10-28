# Mini Lab — Module 02: Switching (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-02.md`

---

## 1. VLAN Plan hoàn chỉnh

| VLAN ID | Tên VLAN | Phòng ban | Switch có access port |
|---|---|---|---|
| 10 | SALES | Sales + Lễ tân | SW-ACC1 |
| 20 | GUEST_WIFI | Guest Wifi | SW-ACC1 |
| 30 | KETOAN_NHANSU | Kế toán + Nhân sự | SW-ACC2 |
| 40 | IT_SERVER | IT + Server | SW-DIST |
| 999 | NATIVE_UNUSED | (không gán host, chỉ để đổi Native VLAN) | Toàn bộ trunk |

## 2. Trunk & Redundancy — quyết định

| Đoạn | Cách xử lý | Lý do |
|---|---|---|
| SW-DIST ↔ SW-ACC1 (2 dây) | Gộp thành **EtherChannel LACP** (Port-channel 1, chế độ `active`/`active`), trunk trên Port-channel | Cả 2 dây cùng forward song song, tăng băng thông thay vì STP khóa chết 1 dây; STP coi Port-channel là 1 link logic nên không có nguy cơ loop. |
| SW-DIST ↔ SW-ACC2 (qua SW-ACC1) | Trunk đơn (Fa0/3–Fa0/4), allowed VLAN không có VLAN 20 | Chỉ 1 dây vật lý, không cần EtherChannel; loại VLAN 20 khỏi allowed list để Guest không lộ sang tầng 2. |

## 3. Cấu hình mẫu

### SW-DIST

```
vlan 10
 name SALES
vlan 20
 name GUEST_WIFI
vlan 30
 name KETOAN_NHANSU
vlan 40
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED
exit

interface range fa0/1-2
 channel-group 1 mode active
 no shutdown

interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,999

! VTP transparent: VLAN cấu hình thủ công từng switch, không để VTP Server tự lan truyền/ghi đè VLAN database
vtp mode transparent

! Access port IT/Server (vi du fa0/10-15)
interface range fa0/10-15
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable

spanning-tree vlan 1,10,20,30,40,999 priority 4096

! Root Guard trên port hướng xuống access: không cho switch khác giành vai trò Root
interface port-channel 1
 spanning-tree guard root
```

### SW-ACC1

```
vlan 10
 name SALES
vlan 20
 name GUEST_WIFI
vlan 30
 name KETOAN_NHANSU
vlan 40
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED
exit

interface range fa0/1-2
 channel-group 1 mode active
 no shutdown

interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,999

! VTP transparent: VLAN cấu hình thủ công từng switch, không để VTP Server tự lan truyền/ghi đè VLAN database
vtp mode transparent

interface fa0/3
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,30,999
! (khong co VLAN 20 trong allowed list xuong SW-ACC2)

interface range fa0/5-9
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable

interface range fa0/20-24
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### SW-ACC2

```
vlan 10
 name SALES
vlan 20
 name GUEST_WIFI
vlan 30
 name KETOAN_NHANSU
vlan 40
 name IT_SERVER
vlan 999
 name NATIVE_UNUSED
exit

! SW-ACC2 không có VLAN 20 (Guest): gỡ VLAN 20 khỏi danh sách VLAN ở trên, không tạo VLAN này trên switch này
no vlan 20
! VTP transparent: VLAN cấu hình thủ công từng switch, không để VTP Server tự lan truyền/ghi đè VLAN database
vtp mode transparent

interface fa0/4
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,30,999

interface range fa0/5-15
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
```

## 4. Output verify mẫu

```
SW-DIST# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------
1      Po1(SU)        LACP       Fa0/1(P) Fa0/2(P)
```
Cả 2 port đều ở trạng thái `(P)` = bundled trong Port-channel, không có port nào bị blocking riêng lẻ.

```
SW-DIST# show spanning-tree vlan 10
VLAN0010
  Root ID    Priority    4106
             This bridge is the root
  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
```

```
SW-ACC1# show vlan brief
VLAN Name          Status  Ports
10   SALES         active  Fa0/5, Fa0/6, ...
20   GUEST_WIFI     active  Fa0/20, Fa0/21, ...
```

## 5. Giải thích các điểm quan trọng / dễ sai

- **EtherChannel phải cấu hình đồng bộ 2 đầu** (cùng mode `active`, cùng danh sách port) — lệch cấu hình 1 đầu sẽ khiến Port-channel không bao giờ lên (`suspended`), đây là lỗi hay gặp nhất khi mới học.
- **Không cấu hình EtherChannel trước, PortFast sau** — PortFast/BPDU Guard chỉ đặt trên access port thật sự nối end-device; nếu lỡ bật trên port đang nối switch khác (kể cả port-channel), BPDU Guard sẽ tự động err-disable port ngay khi nhận BPDU từ switch kia.
- **Native VLAN phải khớp 2 đầu trunk** — nếu SW-DIST để native 999 mà SW-ACC1 còn để mặc định VLAN 1, switch sẽ báo lỗi Native VLAN mismatch (thấy trong log `%CDP-4-NATIVE_VLAN_MISMATCH`).
- **Priority Root Bridge nên đặt tường minh** (ví dụ 4096) thay vì chỉ dùng lệnh `spanning-tree vlan X root primary` — cách tường minh giúp kiểm soát chính xác khi mạng có thêm switch mới với priority thấp hơn mặc định.

> **Lưu ý cập nhật:** Chỉ đặt priority thôi chưa đảm bảo SW-DIST luôn là Root: một switch mới cắm vào với priority thấp hơn (ví dụ 0) vẫn giành được Root. Vì vậy cần đặt priority cho tất cả VLAN đang tồn tại (gồm cả VLAN 1 và 999) và bật Root Guard (`spanning-tree guard root`) trên các port hướng xuống phía access. Với SW-ACC2, VLAN 20 không được tạo (đề bài cấm), nên khối tạo VLAN của SW-ACC2 chỉ giữ VLAN 10/30/40/999.
