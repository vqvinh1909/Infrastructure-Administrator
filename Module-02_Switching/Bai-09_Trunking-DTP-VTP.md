---
title: Bài 9 - Trunking – DTP – VTP
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — VTP là chủ đề có thể gây sự cố toàn mạng nếu cấu hình sai, luôn được hỏi kỹ khi phỏng vấn"
tags: [networking, cisco, trunking, dot1q, dtp, vtp, switching, sysadmin]
aliases: [Trunk, 802.1Q, VTP, Native VLAN]
related: ["[[Bai-08_Cau-hinh-va-quan-ly-VLAN]]", "[[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco]]"]
last_review: 2026-07-21
---

# Bài 9: Trunking – DTP – VTP

> [!important] Tại sao cần file này
> VLAN học ở [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8]] mới chỉ có ý nghĩa trong phạm vi 1 switch. Một doanh nghiệp thực tế luôn có nhiều switch, và cùng một VLAN (ví dụ VLAN Kế toán) thường có máy nằm rải rác trên nhiều switch khác nhau — Trunking là cơ chế cho phép nhiều VLAN cùng đi qua **một đường cáp vật lý duy nhất** giữa các switch. VTP giúp đồng bộ VLAN tự động giữa nhiều switch, nhưng đây cũng là một trong số ít giao thức có thể **xóa sạch VLAN của toàn bộ hệ thống chỉ trong vài giây** nếu cấu hình sai — case study ở mục 8 là lời cảnh báo có thật, không phải giả định.

## 1. Vì sao cần Trunking

Nếu không có trunk, để 2 switch cùng "biết" về 3 VLAN, phải kéo **3 sợi cáp riêng** — mỗi sợi mang đúng 1 VLAN (access port). Trunk giải quyết vấn đề này bằng cách gắn thêm thông tin nhận diện VLAN vào từng frame trước khi gửi qua 1 sợi cáp chung, cho phép switch nhận biết frame đó thuộc VLAN nào dù đi chung đường truyền.

## 2. Cơ chế đóng gói: 802.1Q vs ISL

| | 802.1Q | ISL (Inter-Switch Link) |
|---|---|---|
| Chuẩn | Mở, do IEEE quy định | Độc quyền Cisco |
| Cơ chế | Chèn thêm 4-byte tag vào giữa frame Ethernet gốc | Bọc (encapsulate) toàn bộ frame gốc trong 1 header/trailer mới |
| Overhead | Thấp hơn (chỉ thêm 4 byte) | Cao hơn (thêm 30 byte header + trailer) |
| Hiện trạng | **Chuẩn duy nhất còn dùng thực tế** | Đã lỗi thời, các dòng switch hiện đại không còn hỗ trợ |

```
+----------+----------+------------+----------+--------------+------+
| Dest MAC | Src MAC  | 802.1Q Tag |EtherType |     Data     | FCS  |
+----------+----------+------------+----------+--------------+------+

Chi tiết 802.1Q Tag (4 byte):
+---------------+-----+-----+------------------+
| TPID (0x8100) | PCP | DEI | VLAN ID (12 bit) |
+---------------+-----+-----+------------------+
```

Trường **VLAN ID 12-bit** trong tag chính là lý do dải VLAN lý thuyết tối đa là 4096 (2¹²) — khớp với dải VLAN ID 0-4095 đã học ở mục 2, [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8]].

## 3. Native VLAN

Trên trunk 802.1Q, **mọi VLAN đều được gắn tag — trừ đúng 1 VLAN gọi là Native VLAN, được gửi KHÔNG tag**. Đây là cơ chế thiết kế để tương thích ngược với các thiết bị cũ không hiểu 802.1Q — nếu thiết bị đó nhận được frame không tag, nó sẽ xử lý bình thường như một mạng không có VLAN.

```
Switch(config)# interface gigabitEthernet 0/24
Switch(config-if)# switchport trunk native vlan 999
```

> [!question] Điều gì xảy ra nếu 2 đầu trunk cấu hình Native VLAN khác nhau?
> Đây gọi là **Native VLAN mismatch** — 1 trong những lỗi cấu hình phổ biến và khó phát hiện nhất trên trunk. Vì frame Native VLAN không mang tag, switch nhận sẽ tự động coi nó thuộc về Native VLAN CỦA CHÍNH MÌNH (có thể là VLAN khác với ý định ban đầu) — dẫn đến traffic bị "rò" nhầm sang VLAN khác mà không có lỗi kết nối rõ ràng nào cả (trunk vẫn "up", chỉ có traffic đi sai chỗ). CDP (Bài 7) sẽ cảnh báo bằng log `%CDP-4-NATIVE_VLAN_MISMATCH` nếu phát hiện — đây là lý do 2 đầu trunk luôn phải khớp Native VLAN, không chỉ khớp danh sách VLAN cho phép.

## 4. DTP — Dynamic Trunking Protocol

DTP là giao thức độc quyền Cisco giúp 2 switch tự động thương lượng có nên hình thành trunk hay không, dựa trên mode cấu hình ở mỗi đầu:

| Mode | Hành vi |
|---|---|
| `switchport mode trunk` | Luôn là trunk, có gửi DTP để thông báo (trừ khi thêm `nonegotiate`) |
| `switchport mode access` | Luôn là access, không bao giờ trở thành trunk |
| `switchport mode dynamic desirable` | Chủ động đề nghị hình thành trunk |
| `switchport mode dynamic auto` | Bị động, chỉ trở thành trunk nếu đầu kia chủ động đề nghị |
| `switchport nonegotiate` | Tắt hẳn DTP — không gửi/nhận bản tin đàm phán (dùng kèm `mode trunk` cho cổng trunk cố định) |

**Bảng kết quả khi 2 đầu ở các mode khác nhau:**

| Đầu A \\ Đầu B | Trunk | Access | Dynamic Desirable | Dynamic Auto |
|---|---|---|---|---|
| **Trunk** | Trunk | Lỗi (không khớp) | Trunk | Trunk |
| **Access** | Lỗi | Access | Access | Access |
| **Dynamic Desirable** | Trunk | Access | Trunk | Trunk |
| **Dynamic Auto** | Trunk | Access | Trunk | **Access** (cả 2 cùng bị động, không ai chủ động đề nghị) |

> [!question] Vì sao mục 4 và mục 9 (Bài 8) khuyến nghị luôn cấu hình tường minh `switchport mode access` cho cổng người dùng thay vì để mặc định?
> Vì mặc định trên nhiều dòng switch là `dynamic auto` hoặc `dynamic desirable` — tức là cổng đó **về lý thuyết có thể bị thương lượng thành trunk** nếu thiết bị cắm vào giả làm switch và chủ động gửi DTP đề nghị (đúng kỹ thuật switch spoofing đã học ở Bài 8). Ép cứng `access` + `nonegotiate` loại bỏ hoàn toàn rủi ro này ở các cổng không cần trunk.

## 5. Cấu hình Trunk

```
Switch(config)# interface gigabitEthernet 0/24
Switch(config-if)# switchport trunk encapsulation dot1q
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 999
Switch(config-if)# switchport trunk allowed vlan 10,20,30,99
Switch(config-if)# switchport nonegotiate
```

Lưu ý: lệnh `switchport trunk encapsulation dot1q` chỉ có trên các platform còn hỗ trợ cả ISL (như 3560/3750/4500/6500); trên Catalyst 2960 và Catalyst 9000 chỉ có 802.1Q nên lệnh này bị từ chối — bỏ dòng đó đi, các lệnh còn lại giữ nguyên.

Lệnh `switchport trunk allowed vlan` giới hạn danh sách VLAN thực sự được phép đi qua trunk — mặc định (nếu không cấu hình) là cho phép TẤT CẢ VLAN, nên trong thực tế luôn nên giới hạn tường minh chỉ đúng các VLAN cần thiết, giảm bề mặt tấn công và giảm traffic broadcast không cần thiết lan sang switch không có VLAN đó.

## 6. VTP — VLAN Trunking Protocol

VTP là giao thức giúp đồng bộ **VLAN database** (danh sách VLAN, tên VLAN) tự động giữa nhiều switch trong cùng 1 domain — tránh việc phải gõ tay `vlan X` `name Y` lặp lại trên từng switch riêng lẻ.

| Mode VTP | Có thể tạo/sửa/xóa VLAN? | Có gửi/nhận cập nhật VTP? | Lưu VLAN vào NVRAM? |
|---|---|---|---|
| **Server** | Có | Cả gửi và nhận | Có |
| **Client** | Không (chỉ nhận từ Server) | Chỉ nhận, tự động chuyển tiếp | Tùy phiên bản VTP |
| **Transparent** | Có (nhưng chỉ ảnh hưởng switch đó) | Chuyển tiếp bản tin đi qua nhưng không áp dụng vào database của mình | Có |

## 7. Cấu hình VTP

```
Switch(config)# vtp domain CONGTY-VN
Switch(config)# vtp mode server
Switch(config)# vtp password Vtp@2026
```

Trên các switch còn lại cần đồng bộ, cần đặt đúng **cùng domain name**, **cùng `vtp password`** (nếu server đã đặt password, client thiếu hoặc sai password sẽ từ chối cập nhật), **cùng phiên bản VTP** và có trunk nối giữa các switch, với mode `client` (hoặc `transparent` nếu không muốn tham gia đồng bộ) — không cần tạo lại VLAN tay. VTP có 3 phiên bản (v1/v2/v3); VTPv3 (từ IOS 12.2(52)SE) giảm rủi ro VTP bomb nhờ cơ chế primary server và có thêm mode `off` (`vtp mode off`) để tắt hẳn VTP trên switch hỗ trợ v3.

## 8. Case study — "VTP Bomb": cơ chế Revision Number và hậu quả thật

> [!example] Case study — Sự cố VTP tại một khu học chánh (school district) Mỹ
> Trong một dự án nâng cấp switch, kỹ thuật viên cắm 1 switch mới vào mạng đang chạy VTP mà **không kiểm tra trước VTP configuration revision number** của switch đó. Switch mới này từng thuộc về một domain VTP khác trước đây với revision number **cao hơn** toàn bộ hệ thống hiện tại, nhưng VLAN database bên trong lại **trống**. Theo cơ chế VTP: khi 1 switch nhận bản tin VTP có revision number cao hơn số hiện tại của chính nó, nó **tự động ghi đè toàn bộ VLAN database bằng bản tin nhận được** — bất kể bản tin đó có VLAN hay trống rỗng. Kết quả: toàn bộ VLAN trên mọi switch trong domain bị xóa sạch chỉ trong vài giây, cổng trunk vẫn "up" bình thường nhưng chỉ còn thấy VLAN mặc định — hàng loạt phòng ban mất kết nối dữ liệu, thoại IP, và Wi-Fi cùng lúc. Đội kỹ thuật phải khôi phục thủ công từng switch một, mất nhiều giờ. Một sự cố tương tự khác được ghi nhận ở một mạng campus doanh nghiệp khi thêm switch mới vào lớp Distribution với cùng nguyên nhân gốc.
>
> **Bài học thực chiến**: đây chính là lý do nhiều tổ chức lớn **gần như luôn đặt mọi switch ở chế độ VTP Transparent** hoặc tắt hẳn VTP (`vtp mode off` trên bản hỗ trợ VTPv3), chỉ tạo VLAN tay hoặc qua công cụ tự động hóa (Bài 29-30) — vì rủi ro "1 switch cũ có revision number cao vô tình xóa sạch VLAN toàn mạng" được xem là không tương xứng với lợi ích tiện lợi mà VTP mang lại. Nguồn: [NetBrain — Using Automation to Avoid the VTP Bomb](https://www.netbrain.com/blog/using-automation-avoid-vtp-bomb/), [IPWithEase — What is VTP Bomb](https://ipwithease.com/what-is-vtp-bomb/).

## 9. Quản lý VTP

```
Switch# show vtp status
Switch# show vtp password
Switch(config)# vtp mode transparent
```

**Trước khi thêm bất kỳ switch nào (mới hoặc đã qua sử dụng) vào 1 domain VTP đang chạy Server/Client**, luôn phải kiểm tra `show vtp status` trên switch đó trước — nếu revision number khác 0 hoặc không chắc chắn về lịch sử switch, cách an toàn nhất là đặt tạm `vtp mode transparent`, hoặc đổi `vtp domain` sang một tên bất kỳ rồi đổi lại đúng tên domain thật (thao tác này tự động reset revision number về 0) trước khi kết nối vào mạng thật.

**Xóa một switch khỏi domain VTP** (không muốn nó tiếp tục đồng bộ VLAN database với domain hiện tại nữa) — có 2 cách, tùy mục đích:

- `vtp mode transparent`: switch vẫn nhận và chuyển tiếp bản tin VTP đi qua nó, nhưng **không áp dụng** thay đổi vào VLAN database riêng — về hiệu quả là "ngắt" khỏi việc bị đồng bộ, dù về mặt kỹ thuật vẫn thuộc domain đó.
- Đổi hẳn `vtp domain <tên khác>`: vì switch chỉ đồng bộ với các switch **cùng tên domain**, đổi sang tên domain khác (hoặc domain rỗng nếu switch hỗ trợ) sẽ ngắt hoàn toàn switch đó khỏi domain cũ. Thao tác này đồng thời reset revision number về 0 — đây cũng là lý do nó được dùng làm bước chuẩn bị an toàn trước khi đưa switch quay lại tham gia domain khác (đã nêu ở đoạn trên).

## 10. Verify & Troubleshoot

| Lệnh | Công dụng |
|---|---|
| `show interfaces trunk` | Liệt kê các cổng đang ở trạng thái trunk, VLAN allowed, Native VLAN |
| `show vtp status` | Xem mode VTP, domain, revision number hiện tại |
| `show cdp neighbors detail` | Kiểm tra cảnh báo Native VLAN mismatch nếu có |

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Trunk lên nhưng VLAN vẫn không thông giữa 2 switch | Thiếu VLAN đó trong `switchport trunk allowed vlan` ở 1 hoặc cả 2 đầu | `show interfaces trunk`, đối chiếu danh sách allowed VLAN 2 đầu |
| Traffic rò nhầm VLAN dù không báo lỗi trunk | Native VLAN mismatch 2 đầu | Kiểm tra log CDP, đồng bộ lại Native VLAN 2 đầu |
| Toàn bộ VLAN biến mất sau khi thêm switch mới | VTP Bomb — switch mới có revision number cao hơn, database trống | Khôi phục VLAN thủ công, sau đó chuyển toàn bộ domain sang Transparent hoặc kiểm soát chặt quy trình thêm switch mới |
| 2 cổng cùng cấu hình `dynamic auto` không lên trunk | Cả 2 đều bị động, không ai chủ động đề nghị (đúng theo bảng DTP mục 4) | Đổi ít nhất 1 đầu sang `dynamic desirable` hoặc `trunk` tường minh |

## 11. Lab thực hành

**Môi trường**: Cisco Packet Tracer, tối thiểu 2 switch.

1. Cấu hình trunk 802.1Q giữa 2 switch theo mục 5, đặt cùng Native VLAN 999 ở cả 2 đầu.
2. Tạo cùng VLAN 10/20 ở cả 2 switch (thủ công, chưa dùng VTP), gán mỗi switch 1 PC cho mỗi VLAN, xác nhận 2 PC cùng VLAN nhưng khác switch vẫn ping thông qua trunk.
3. Cố tình đổi Native VLAN 1 đầu thành 1 (khác đầu kia), quan sát log cảnh báo `NATIVE_VLAN_MISMATCH` qua `show cdp neighbors detail` (nếu Packet Tracer mô phỏng được) hoặc `show interfaces trunk` chỉ ra sự khác biệt.
4. Thiết lập VTP domain + server/client giữa 2 switch, tạo 1 VLAN mới trên Server, xác nhận VLAN tự động xuất hiện trên Client mà không cần gõ tay — sau đó thử đổi `vtp mode transparent` trên switch Client và tạo VLAN mới trên Server, quan sát VLAN đó **không** xuất hiện trên switch Transparent (đúng hành vi mục 6).

**Kết quả kỳ vọng**: hiểu và thực hành được toàn bộ vòng đời trunk từ cấu hình cơ bản đến các lỗi thực chiến (Native VLAN mismatch, VTP đồng bộ/không đồng bộ theo mode).

## 12. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Trunk tự động cho phép mọi VLAN đi qua, không cần cấu hình gì thêm | Đúng về mặc định kỹ thuật nhưng không an toàn | Nên luôn giới hạn tường minh bằng `switchport trunk allowed vlan` |
| VTP luôn nên bật vì "tiện đồng bộ tự động" | Bỏ qua rủi ro VTP Bomb đã học ở mục 8 | Nhiều tổ chức chủ động chọn Transparent hoặc tắt VTP để loại bỏ rủi ro ghi đè toàn bộ VLAN database |
| Native VLAN mismatch sẽ làm trunk "down" ngay, dễ phát hiện | Nhầm lẫn với lỗi encapsulation không khớp (lỗi này mới làm trunk down) | Native VLAN mismatch trunk vẫn "up" bình thường, chỉ traffic đi sai VLAN — khó phát hiện hơn nhiều |
| ISL vẫn là lựa chọn tương đương 802.1Q, tùy sở thích | Nhầm lẫn 2 chuẩn cùng "làm trunk" là ngang hàng | ISL đã lỗi thời, thiết bị hiện đại chỉ còn hỗ trợ 802.1Q |

## 13. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Trunk khác Access port ở điểm nào? Vì sao cần trunk giữa các switch?**
→ Access port chỉ mang đúng 1 VLAN, không gắn tag. Trunk mang được nhiều VLAN qua cùng 1 đường cáp bằng cách gắn tag 802.1Q phân biệt từng VLAN. Cần trunk vì 1 VLAN thường có máy nằm trên nhiều switch khác nhau, không thể kéo riêng 1 sợi cáp cho mỗi VLAN.

**2. Native VLAN là gì? Vì sao mismatch giữa 2 đầu lại nguy hiểm?**
→ Native VLAN là VLAN duy nhất được gửi KHÔNG gắn tag trên trunk 802.1Q (để tương thích thiết bị cũ). Nếu 2 đầu cấu hình Native VLAN khác nhau, switch nhận sẽ hiểu nhầm frame không tag thuộc Native VLAN của chính nó — traffic bị rò sang VLAN sai mà trunk vẫn hoạt động bình thường, không có cảnh báo lỗi rõ ràng.

**3. Trình bày cơ chế gây ra "VTP Bomb" và cách phòng tránh.**
→ Khi 1 switch VTP Server/Client nhận bản tin VTP có revision number cao hơn số hiện tại của nó, nó tự động ghi đè toàn bộ VLAN database — kể cả khi bản tin đó trống. Một switch cũ mang revision number cao cắm nhầm vào mạng có thể xóa sạch VLAN toàn hệ thống. Phòng tránh: kiểm tra `show vtp status` trước khi thêm switch mới, hoặc dùng VTP Transparent/tắt VTP hoàn toàn.

**4. 2 cổng ở mode `dynamic auto` cả 2 đầu có hình thành trunk không? Vì sao?**
→ Không. Cả 2 đều ở trạng thái bị động, chỉ phản hồi khi có bên chủ động đề nghị (`desirable` hoặc `trunk`) — không ai chủ động nên không bên nào khởi tạo trunk, kết quả là Access.

**5. Vì sao `switchport trunk allowed vlan` nên luôn được cấu hình tường minh?**
→ Vì mặc định trunk cho phép TẤT CẢ VLAN đi qua, làm tăng bề mặt tấn công (VLAN không liên quan vẫn có traffic broadcast lan tới) và không có kiểm soát rõ ràng VLAN nào thực sự cần thiết trên từng liên kết.

## 14. Tài liệu tham khảo

- [Cisco — Configure 802.1Q Trunking Between Catalyst Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-6000-series-switches/10599-88.html)
- [NetworkLessons — Cisco DTP (Dynamic Trunking Protocol) Negotiation](https://networklessons.com/switching/cisco-dtp-dynamic-trunking-protocol-negotiation)
- [NetBrain — Using Automation to Avoid the VTP Bomb](https://www.netbrain.com/blog/using-automation-avoid-vtp-bomb/)
- [IPWithEase — What is VTP Bomb](https://ipwithease.com/what-is-vtp-bomb/)

---
**Liên kết**: [[Bai-08_Cau-hinh-va-quan-ly-VLAN|← Bài 8]] · [[Bai-10_STP-va-EtherChannel|Bài 10 - STP và EtherChannel →]]
