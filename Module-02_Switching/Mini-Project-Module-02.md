# Mini Lab — Module 02: Switching (CCNA)

> Áp dụng sau khi hoàn thành Bài 07–10.
> File: `Mini-Project-Module-02.md` · Lời giải: `Mini-Project-Module-02-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Bối cảnh doanh nghiệp

Tiếp nối Module 01 — **TechStart JSC** đã đi cáp xong, giờ đội thi công vừa lắp thêm **2 sợi cáp dự phòng** giữa Switch trung tâm (Server Room) và Switch tầng 1 (Sales/Guest), phòng trường hợp đứt cáp chính. Trưởng nhóm IT dặn: "Đấu thêm cáp dự phòng nhưng đừng để 2 sợi đó tạo thành vòng lặp làm sập cả mạng — tuần trước bên đối tác bị y chang vậy, switch bị broadcast storm treo cứng cả văn phòng."

Bạn cần dựng hạ tầng switching hoàn chỉnh: VLAN theo phòng ban, trunk giữa các switch, xử lý đúng 2 link dự phòng, và đảm bảo switch trung tâm luôn là điểm ra quyết định của Spanning Tree.

## 2. Sơ đồ mạng (đề bài)

```
[ SW-DIST ]  (Switch trung tam, Server Room)
    ||
    || (Fa0/1 va Fa0/2 - 2 day cap du phong toi cung 1 switch tang 1)
    ||
[ SW-ACC1 ]  (Switch tang 1: Sales va Guest)
    |
    | (Fa0/3 - 1 day toi SW-ACC2)
    |
[ SW-ACC2 ]  (Switch tang 2: Ke toan va Nhan su)
```

## 3. Bảng cần điền

**a) VLAN Plan** (dựa trên bảng VLSM đã tính ở Module 01):

| VLAN ID | Tên VLAN | Phòng ban | Switch nào có access port |
|---|---|---|---|
| 10 | | Sales + Lễ tân | |
| 20 | | Guest Wifi | |
| 30 | | Kế toán + Nhân sự | |
| 40 | | IT + Server | |

**b) Trunk & Redundancy Plan** — điền cách xử lý cho từng đoạn:

| Đoạn | Số dây vật lý | Cách xử lý đề xuất | Lý do |
|---|---|---|---|
| SW-DIST ↔ SW-ACC1 | 2 (Fa0/1, Fa0/2) | | |
| SW-DIST ↔ SW-ACC2 | 1 (qua SW-ACC1) | | |

## 4. Yêu cầu kỹ thuật

- [ ] Tạo đủ VLAN đúng ID/tên trên các switch (riêng VLAN 20 Guest chỉ tạo trên SW-DIST và SW-ACC1, không tạo trên SW-ACC2), gán access port đúng VLAN theo phòng ban.
- [ ] Cấu hình trunk cho các đường liên kết switch-to-switch, đổi Native VLAN sang 1 VLAN không dùng cho user (không để mặc định VLAN 1).
- [ ] Xử lý 2 dây dự phòng giữa SW-DIST và SW-ACC1 sao cho: (1) không gây vòng lặp, (2) vẫn tận dụng được cả 2 dây thay vì để 1 dây bị STP khóa hoàn toàn không dùng tới.
- [ ] SW-DIST phải luôn là Root Bridge cho toàn bộ VLAN — kể cả khi có switch khác gắn thêm vào mạng sau này.
- [ ] Toàn bộ access port nối máy người dùng (không phải port nối switch khác) phải chuyển sang trạng thái forwarding nhanh và có cơ chế chặn nếu ai đó cắm nhầm 1 switch khác vào port này.

## 5. Ràng buộc

- Không được gỡ bỏ vật lý 1 trong 2 dây dự phòng — bài toán là xử lý bằng cấu hình, không phải bằng cách rút cáp.
- Guest Wifi (VLAN 20) không được xuất hiện trên SW-ACC2 (không có nhu cầu, giảm diện tấn công).
- Không cấu hình VTP ở chế độ Server lan truyền tự động giữa các switch trong bài này — quyết định VLAN thủ công trên từng switch, tự giải thích lý do chọn VTP mode nào (kể cả nếu chọn không dùng VTP).

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| VLAN | Đủ 4 VLAN, access port đúng phòng ban, Guest không lộ sang SW-ACC2 | Thiếu VLAN, gán nhầm port, hoặc VLAN 20 xuất hiện ở SW-ACC2 |
| Trunk | Trunk lên đúng giữa các switch, Native VLAN đã đổi khỏi VLAN 1 | Còn để Native VLAN 1, hoặc trunk không truyền đủ VLAN cần thiết |
| Redundancy 2 dây | Dùng được cả 2 dây song song, không có vòng lặp, xác minh bằng show command | Chỉ dùng 1 dây (dây kia bị STP khóa im), hoặc gây loop |
| Root Bridge | SW-DIST có priority thấp nhất, xác minh bằng `show spanning-tree` | Không set priority, phụ thuộc bầu chọn ngẫu nhiên theo MAC |
| Access port protection | Có PortFast + BPDU Guard trên port nối end-device | Thiếu 1 trong 2, hoặc bật nhầm trên port nối switch |

## 7. Gợi ý triển khai

1. Với 2 dây dự phòng giữa SW-DIST và SW-ACC1, xem lại khái niệm gộp nhiều link vật lý thành 1 logical link ở Bài 10 — đây chính là công cụ giải quyết đúng bài toán "dùng được cả 2 dây mà không loop".
2. Set priority thủ công cho SW-DIST thay vì để mặc định 32768 — nhớ lại công thức Bridge ID (Bài 10).
3. Xác định trước native VLAN mới (ví dụ 1 VLAN "quản lý"/"đen", chưa dùng) trước khi cấu hình trunk.
4. Cấu hình PortFast/BPDU Guard chỉ trên port đã xác định chắc chắn là access port nối thiết bị đầu cuối, không áp lên port trunk.

## 8. Lệnh verify cần chạy

- `show vlan brief` trên cả 3 switch — kiểm tra VLAN và port gán đúng.
- `show interfaces trunk` — kiểm tra trunk lên, Native VLAN, danh sách VLAN allowed.
- `show etherchannel summary` (hoặc lệnh tương ứng) — xác nhận 2 dây dự phòng đang hoạt động cùng lúc, không bị 1 dây ở trạng thái blocking.
- `show spanning-tree` — xác nhận SW-DIST là Root cho mọi VLAN.
- `show spanning-tree interface fa0/x portfast` trên access port — xác nhận PortFast đang bật.

## 9. Kết quả kỳ vọng

Hệ thống 3 switch chạy VLAN đúng theo phòng ban, trunk ổn định giữa các switch với Native VLAN đã đổi, 2 dây dự phòng giữa SW-DIST/SW-ACC1 được tận dụng đồng thời mà không gây vòng lặp, SW-DIST luôn giữ vai trò Root Bridge, và các access port có bảo vệ PortFast/BPDU Guard — sẵn sàng để Module 03 nối tiếp phần định tuyến giữa các VLAN.

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (VLAN name cụ thể, cấu hình EtherChannel/STP) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-02-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 08 (VLAN), Bài 09 (Trunk/Native VLAN/VTP), Bài 10 (STP Root Bridge, EtherChannel, PortFast/BPDU Guard).
