---
title: Bài 8 - Cấu hình và quản lý VLAN
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — VLAN là kỹ thuật switching nền tảng nhất, xuất hiện trong gần như mọi bài test thực hành"
tags: [networking, cisco, vlan, switching, sysadmin]
aliases: [VLAN, Virtual LAN]
related: ["[[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco]]", "[[Bai-01_Gioi-thieu-mang-may-tinh]]"]
last_review: 2026-07-21
---

# Bài 8: Cấu hình và quản lý VLAN

> [!important] Tại sao cần file này
> [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]] (mục 5.2) đã giải thích vì sao doanh nghiệp không thể dùng 1 switch phẳng cho cả tòa nhà — vì miền broadcast quá lớn và không phân vùng được theo phòng ban. VLAN chính là công cụ hiện thực hóa việc phân vùng đó **ngay trên cùng một switch vật lý**, không cần mua thêm thiết bị. Đây là kỹ thuật switching cơ bản nhất và xuất hiện trong hầu như mọi bài test thực hành phỏng vấn.

## 1. VLAN là gì và mục đích

**VLAN (Virtual LAN)** là một broadcast domain logic được tạo ra bên trong switch, độc lập với vị trí vật lý của cổng. Về bản chất, VLAN chia 1 switch vật lý thành nhiều switch logic tách biệt — 2 cổng cùng switch nhưng khác VLAN sẽ **không** thấy broadcast của nhau, giống như đang cắm vào 2 switch hoàn toàn riêng biệt (nhắc lại khái niệm broadcast domain đã học ở mục 6, [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]).

**Lợi ích cốt lõi:**
- **Kiểm soát broadcast domain**: giới hạn phạm vi lan truyền broadcast, giảm tải không cần thiết lên toàn mạng.
- **Bảo mật/phân vùng**: tách biệt traffic giữa các phòng ban (kế toán không "nghe" được traffic của kỹ thuật) mà không cần switch vật lý riêng.
- **Linh hoạt theo tổ chức, không theo vị trí**: một nhân viên chuyển chỗ ngồi khác tầng vẫn có thể giữ nguyên VLAN cũ chỉ bằng cách gán lại đúng VLAN cho cổng mới — không phụ thuộc dây cáp vật lý.

## 2. Các VLAN mặc định và dải số hiệu

| Dải VLAN ID | Tên gọi | Ghi chú |
|---|---|---|
| 0, 4095 | Reserved (hệ thống) | Không dùng được |
| **1** | Default VLAN | Tồn tại sẵn, **không xóa/đổi tên được**, mọi cổng switch mặc định thuộc VLAN 1 nếu chưa cấu hình gì |
| 2 – 1001 | Normal Range | Dải VLAN thường dùng để quy hoạch thực tế |
| 1002 – 1005 | Reserved (Token Ring/FDDI) | Tự động tồn tại sẵn, không dùng cho Ethernet hiện đại, không xóa được |
| 1006 – 4094 | Extended Range | Dùng khi cần nhiều VLAN hơn dải Normal (ít gặp ở doanh nghiệp vừa/nhỏ) |

> [!question] VLAN 1 có nên dùng làm VLAN dữ liệu chính không?
> Không khuyến nghị. Vì VLAN 1 là mặc định của mọi cổng chưa cấu hình và cũng thường là Native VLAN mặc định (chi tiết ở Bài 9), việc dùng VLAN 1 cho dữ liệu thật khiến các traffic quản lý mặc định (CDP, VTP, STP BPDU...) trộn lẫn với traffic người dùng, tăng bề mặt tấn công. Thực hành chuẩn: đổi Native VLAN sang một VLAN riêng không dùng cho dữ liệu, và không gán VLAN 1 cho bất kỳ port dữ liệu thật nào.

## 3. Quy hoạch VLAN cho doanh nghiệp

Quy hoạch VLAN thường theo 2 tiêu chí kết hợp: **theo phòng ban/chức năng** và **theo loại traffic**. Ví dụ quy hoạch cho công ty ở case study Bài 1 (mục 5.4):

| VLAN ID | Tên | Mục đích |
|---|---|---|
| 10 | KINH_DOANH | Dữ liệu phòng Kinh doanh |
| 20 | KY_THUAT | Dữ liệu phòng Kỹ thuật |
| 30 | KE_TOAN | Dữ liệu phòng Kế toán |
| 99 | MGMT | Quản trị thiết bị (switch/router/firewall) — tách riêng khỏi VLAN người dùng |
| 100 | VOICE | Điện thoại IP (thường tách riêng vì yêu cầu QoS khác dữ liệu thường) |
| 999 | NATIVE_UNUSED | Native VLAN "rác" không gán cho port nào — chỉ để không dùng VLAN 1 làm Native (mục 2) |

## 4. Cấu hình VLAN cơ bản

```
Switch> enable
Switch# configure terminal
Switch(config)# vlan 10
Switch(config-vlan)# name KINH_DOANH
Switch(config-vlan)# exit
Switch(config)# vlan 20
Switch(config-vlan)# name KY_THUAT
Switch(config-vlan)# exit
```

**Gán cổng vào VLAN (access port):**

```
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# exit
```

> [!question] Vì sao cần cả `switchport mode access` lẫn `switchport access vlan`, thiếu 1 lệnh có sao không?
> `switchport mode access` ép cổng luôn ở chế độ Access (không tự thương lượng trunk qua DTP — liên quan Bài 9), còn `switchport access vlan` mới thực sự gán VLAN cụ thể cho cổng đó. Thiếu `switchport mode access`, cổng vẫn hoạt động nhưng chế độ mặc định trên Catalyst hiện đại là `dynamic auto` (một số dòng cũ là `dynamic desirable`), không phải Access — tức cổng ở chế độ tự động thương lượng DTP tiềm ẩn rủi ro bị lợi dụng qua kỹ thuật switch spoofing (mục 9) — nên luôn cấu hình tường minh cả 2 lệnh thay vì dựa vào default.

## 5. Gán nhiều cổng cùng lúc bằng interface range

```
Switch(config)# interface range gigabitEthernet 0/1 - 10
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 10
Switch(config-if-range)# exit
```

## 6. Xóa VLAN và chuyển port sang VLAN khác

```
Switch(config)# no vlan 20
```

> [!question] Điều gì xảy ra với các cổng đang gán VLAN 20 sau khi xóa VLAN đó?
> Các cổng đó **không tự động chuyển về VLAN 1** — chúng rơi vào trạng thái **inactive** (mất kết nối) vì VLAN được gán không còn tồn tại, dù cấu hình `switchport access vlan 20` trên từng cổng vẫn còn nguyên trong running-config. Đây là lỗi thực chiến rất dễ gặp: xóa nhầm VLAN đang có người dùng thật sẽ gây mất kết nối hàng loạt ngay lập tức mà không có cảnh báo trước. Phải gán lại cổng sang VLAN khác (`switchport access vlan <id mới>`) để khôi phục.

**Chuyển port từ VLAN này sang VLAN khác** — chỉ cần chạy lại lệnh gán VLAN mới, không cần xóa gán cũ trước:

```
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# switchport access vlan 30
```

## 7. Kiểm tra (Verify)

| Lệnh | Công dụng |
|---|---|
| `show vlan brief` | Xem toàn bộ danh sách VLAN và các cổng access đang thuộc từng VLAN |
| `show interfaces gigabitEthernet 0/1 switchport` | Xem chi tiết trạng thái switchport của 1 cổng cụ thể (mode, VLAN access, VLAN voice nếu có) |
| `show running-config interface gigabitEthernet 0/1` | Xem đúng các lệnh đã cấu hình cho 1 cổng |

## 8. Các lỗi thường gặp liên quan đến VLAN và cách xử lý

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| PC không ping được PC khác dù cùng VLAN | Sai `switchport access vlan` (gán nhầm VLAN), hoặc cổng đang ở trạng thái `shutdown`/inactive | `show interfaces switchport` đối chiếu đúng VLAN đang gán; `show vlan brief` xác nhận cổng nằm đúng danh sách VLAN mong muốn |
| Cổng hiển thị VLAN đúng nhưng vẫn không lên (down) | VLAN đó đã bị xóa (`no vlan X`) trong khi cổng vẫn còn gán vào nó — cổng rơi vào trạng thái inactive (mục 6) | Gán lại cổng sang 1 VLAN đang thực sự tồn tại (`switchport access vlan <id còn tồn tại>`) |
| Tạo VLAN nhưng không thấy trong `show vlan brief` | Gõ nhầm số VLAN, hoặc đang thao tác nhầm trên switch khác trong lúc quản lý nhiều thiết bị cùng lúc | Chạy lại `show vlan brief` ngay trên đúng switch, đối chiếu số VLAN vừa tạo bằng `show running-config | section vlan` |
| 2 PC cùng VLAN nhưng khác switch không ping được nhau | Thiếu cấu hình trunk giữa 2 switch, hoặc VLAN đó chưa được cho phép qua trunk (`switchport trunk allowed vlan`) | Xem trước ở [[Bai-09_Trunking-DTP-VTP|Bài 9]] — đây là lý do VLAN một mình chưa đủ, cần Trunking để mở rộng VLAN xuyên nhiều switch |
| Không thể đổi tên hoặc xóa VLAN 1 / VLAN 1002-1005 | Đây là các VLAN hệ thống mặc định (mục 2), Cisco IOS chặn thao tác này | Không có cách xử lý — đây là hành vi thiết kế đúng, không phải lỗi cần khắc phục |

## 9. Bảo mật liên quan VLAN — VLAN Hopping

Hai kỹ thuật tấn công phổ biến khai thác cấu hình VLAN lỏng lẻo, cần biết để tránh khi cấu hình thực tế (chi tiết hardening đầy đủ ở Bài 27):

- **Switch spoofing**: kẻ tấn công cấu hình máy của mình giả làm switch, lợi dụng DTP (Dynamic Trunking Protocol — Bài 9) để thương lượng trunk với switch thật. Nếu thành công, kẻ tấn công có được quyền truy cập **2 chiều** vào mọi VLAN đi qua trunk đó. Phòng chống: luôn ép cổng người dùng ở chế độ Access tường minh (mục 4) thay vì để mặc định tự thương lượng.
- **Double tagging**: kẻ tấn công gửi frame gắn **2 lớp tag 802.1Q** — tag ngoài trùng Native VLAN, tag trong là VLAN muốn xâm nhập. Switch đầu tiên bóc tag ngoài (vì trùng Native VLAN nên xử lý như frame không gắn tag) rồi chuyển tiếp tiếp frame còn tag trong sang switch kế tiếp — vô tình đưa frame vào đúng VLAN mục tiêu. Kỹ thuật này thường chỉ gửi được 1 chiều (không nhận được phản hồi) nhưng vẫn đủ nguy hiểm cho các cuộc tấn công không cần phản hồi hai chiều. Phòng chống: không bao giờ dùng VLAN 1 (hoặc bất kỳ VLAN đang dùng cho dữ liệu) làm Native VLAN — đây chính là lý do khuyến nghị VLAN 999 "rác" ở mục 3.

## 10. Lab thực hành

**Môi trường**: Cisco Packet Tracer.

1. Tạo 3 VLAN (10, 20, 30) với tên tương ứng theo mục 3, gán mỗi VLAN cho 2 PC khác nhau trên cùng 1 switch.
2. Xác nhận 2 PC cùng VLAN ping thông nhau, 2 PC khác VLAN **không** ping thông (vì chưa có định tuyến liên VLAN — nội dung Bài 12).
3. Chạy `show vlan brief`, đối chiếu đúng danh sách cổng theo từng VLAN.
4. Cố tình `no vlan 20` khi đang có PC gán VLAN đó, quan sát PC mất kết nối (`show interfaces switchport` sẽ báo VLAN không tồn tại) — sau đó gán lại đúng VLAN để khôi phục, ghi nhớ bài học từ mục 6.

**Kết quả kỳ vọng**: hiểu và chứng minh được bằng thực hành rằng VLAN tạo ra ranh giới broadcast domain thật sự trên cùng 1 switch vật lý, và nắm được hậu quả cụ thể khi thao tác sai (xóa VLAN đang dùng).

## 11. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Cùng switch vật lý thì luôn liên lạc được với nhau | Bỏ qua khái niệm VLAN tạo broadcast domain riêng | 2 cổng khác VLAN trên cùng switch không liên lạc trực tiếp được nếu không có định tuyến liên VLAN (Bài 12) |
| Xóa VLAN sẽ tự động trả các cổng về VLAN 1 | Nhầm lẫn với hành vi "reset về mặc định" | Các cổng rơi vào trạng thái inactive, phải gán tay VLAN mới để khôi phục |
| VLAN 1 dùng cho dữ liệu cũng an toàn như VLAN khác vì "vẫn là VLAN riêng" | Bỏ qua việc VLAN 1 thường trùng Native VLAN mặc định, mang theo rủi ro double tagging | Nên tránh dùng VLAN 1 cho dữ liệu thật, đổi Native VLAN sang VLAN không dùng |
| Chỉ cần `switchport access vlan X` là đủ, không cần `switchport mode access` | Cho rằng 1 lệnh đã đủ chỉ định VLAN | Nên cấu hình tường minh cả 2 lệnh để tránh cổng ở chế độ tự thương lượng trunk, giảm rủi ro switch spoofing |

## 12. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. VLAN giải quyết vấn đề gì mà switch thông thường không giải quyết được?**
→ Switch thông thường tạo nhiều collision domain (mỗi port riêng) nhưng vẫn chỉ có 1 broadcast domain duy nhất trên toàn switch. VLAN chia nhỏ broadcast domain đó thành nhiều vùng logic độc lập ngay trên cùng phần cứng vật lý.

**2. Vì sao VLAN 1002-1005 tồn tại mặc định và không xóa được?**
→ Đây là các VLAN dành riêng cho Token Ring/FDDI theo thiết kế lịch sử của Cisco, tự động tồn tại trên switch dù không còn dùng cho Ethernet hiện đại — thuộc nhóm VLAN hệ thống không thể xóa.

**3. Điều gì xảy ra nếu xóa một VLAN đang có cổng gán vào nó?**
→ Các cổng đó không tự chuyển VLAN nào khác, rơi vào trạng thái inactive/mất kết nối cho đến khi được gán tay sang VLAN khác đang tồn tại.

**4. Switch spoofing và double tagging khác nhau ở điểm nào?**
→ Switch spoofing khai thác DTP để tự thương lượng trở thành trunk, cho phép truy cập 2 chiều mọi VLAN qua trunk đó. Double tagging gắn 2 lớp tag 802.1Q lợi dụng Native VLAN để "lách" qua switch đầu tiên vào VLAN mục tiêu, thường chỉ gửi được 1 chiều.

**5. Vì sao nên đổi Native VLAN sang một VLAN không dùng cho dữ liệu thay vì giữ mặc định VLAN 1?**
→ Vì kỹ thuật double tagging lợi dụng đúng Native VLAN để bóc tag ngoài — nếu Native VLAN trùng VLAN dữ liệu thật (mặc định là VLAN 1), kẻ tấn công có thể lách vào VLAN đó. Dùng một VLAN "rác" không gán cho port dữ liệu nào làm Native VLAN loại bỏ được rủi ro này.

## 13. Tài liệu tham khảo

- [Cisco — VLAN Configuration Guide (Catalyst 9000)](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/vlan/vlan-configuration-guide/configure-vlan.html)
- [Imperva — What is VLAN Hopping: Risks, Attacks & Prevention](https://www.imperva.com/learn/availability/vlan-hopping/)
- [NetworkLessons — VLAN Configuration on Cisco IOS Switch](https://networklessons.com/switching/how-to-configure-vlans-on-cisco-catalyst-switch)

---
**Liên kết**: [[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco|← Bài 7]] · [[Bai-09_Trunking-DTP-VTP|Bài 9 - Trunking – DTP – VTP →]]
