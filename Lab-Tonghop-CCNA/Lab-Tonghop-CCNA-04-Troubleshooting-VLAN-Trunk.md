# Lab Tổng Hợp CCNA 04 — Troubleshooting Ticket: VLAN/Trunk sau khi thay switch (Module 02)

> Kết hợp trọng tâm: Module 02 (Switching — VLAN/Trunk).
> File: `Lab-Tonghop-CCNA-04-Troubleshooting-VLAN-Trunk.md` · Lời giải: `Lab-Tonghop-CCNA-04-Troubleshooting-VLAN-Trunk-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Ticket từ Help Desk

> **Ticket #4471 — Ưu tiên: Cao**
> Sáng nay đội hạ tầng thay switch tầng Kế toán (SW-EDGE) bằng thiết bị mới do switch cũ hỏng nguồn. Sau khi thay xong, toàn bộ máy tính phòng Kế toán không vào được mạng — máy báo "Không có Internet", thử gán tay 1 IP trong đúng dải Kế toán cũng không ping được gateway. Các phòng ban khác (đấu vào switch khác) vẫn hoạt động bình thường. Đội thi công khẳng định "đã copy y nguyên cấu hình switch cũ sang switch mới rồi".

## 2. Sơ đồ mạng liên quan

```
                 +-------------+
                 |   SW-CORE   |  (VLAN 10 SALES, 20 GUEST, 30 KETOAN, 40 IT da ton tai san)
                 +------+------+
                        | Fa0/1 (trunk)
                 +------+------+
                 | SW-EDGE-NEW |  (vua thay, phong Ke toan - VLAN 30)
                 +------+------+
                        | Fa0/5 (access)
                    [ PC Ke toan ]
```

## 3. Cấu hình hiện tại trên SW-EDGE-NEW (trích, có lỗi)

```
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,40

interface FastEthernet0/5
 switchport mode access
 switchport access vlan 30
 no shutdown
```

`show vlan brief` trên SW-EDGE-NEW chỉ liệt kê VLAN 1 và các VLAN dành riêng 1002–1005, không thấy VLAN 30 trong danh sách.

> **Lưu ý cập nhật:** giả định của bài là SW-EDGE-NEW mới đang ở chế độ VTP client (hoặc VLAN 30 chưa được tạo) nên IOS không tự tạo VLAN 30 khi gõ `switchport access vlan 30`; ở chế độ server/transparent, IOS sẽ tự tạo VLAN này.

## 4. Yêu cầu chẩn đoán

- [ ] Xác định **tất cả** nguyên nhân gây mất kết nối (gợi ý: có nhiều hơn 1 nguyên nhân, sửa 1 chỗ chưa chắc đã hết lỗi).
- [ ] Sửa lỗi mà **không** cấu hình lại toàn bộ switch từ đầu — chỉ sửa đúng chỗ sai.
- [ ] Xác nhận lại bằng lệnh `show` cụ thể sau khi sửa, không chỉ dựa vào "PC lên mạng là xong".
- [ ] Viết ngắn gọn (không cần file riêng) lý do vì sao "copy y nguyên cấu hình" từ switch cũ vẫn có thể gây lỗi này — đây là bài học quan trọng hơn cả việc sửa lỗi.

## 5. Ràng buộc

- Không được đổi VLAN ID hay đổi port đang dùng — sửa đúng bằng cấu hình, không "né" lỗi bằng cách đổi sang VLAN/port khác.
- Không factory-reset SW-EDGE-NEW rồi cấu hình lại từ đầu — đây là bài luyện chẩn đoán, không phải luyện gõ lại cấu hình.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Xác định đủ nguyên nhân | Tìm ra cả 2 lỗi liên quan tới VLAN 30 (không chỉ sửa 1 chỗ rồi dừng) | Chỉ sửa 1 lỗi, PC vẫn không lên mạng nhưng báo "đã xong" |
| Cách sửa | Sửa đúng, tối thiểu, không đổi cấu trúc không liên quan | Cấu hình lại toàn bộ hoặc đổi VLAN/port khác đi |
| Xác minh | Có `show` command cụ thể chứng minh đã hết lỗi | Chỉ nói "chạy được rồi" không có bằng chứng |
| Giải thích nguyên nhân | Lý giải đúng vì sao VTP/VLAN cục bộ khiến "copy y nguyên" không đủ | Không giải thích được hoặc giải thích sai bản chất |

## 7. Gợi ý cách tiếp cận (không phải đáp án)

1. Đi từ Lớp 1 lên: kiểm tra port có `up/up` chưa, trước khi nghi ngờ VLAN/trunk.
2. `show vlan brief` là lệnh đầu tiên nên chạy khi nghi ngờ vấn đề VLAN — nó cho biết ngay VLAN có tồn tại trên switch đó hay không, và port nào đang thuộc VLAN nào.
3. Đừng dừng lại ngay sau khi tìm thấy 1 lỗi — với 1 ticket ghi "toàn bộ phòng ban mất mạng", thường có nhiều hơn 1 điểm cấu hình bị bỏ sót khi thay thiết bị.
4. Nhớ lại quyết định ở Mini Project Module 02: công ty **không dùng VTP Server** — điều này có hệ quả gì với switch mới toanh vừa thay vào?

## 8. Lệnh chẩn đoán cần chạy

- `show interfaces status` — kiểm tra trạng thái vật lý port Fa0/5.
- `show vlan brief` — kiểm tra VLAN 30 có tồn tại và port Fa0/5 có thuộc đúng VLAN.
- `show interfaces trunk` — kiểm tra Fa0/1 có đang là trunk và VLAN nào được phép đi qua.
- `show running-config interface fa0/5` và `fa0/1` — đối chiếu với cấu hình gốc đã cung cấp.

## 9. Kết quả kỳ vọng sau khi sửa

PC phòng Kế toán nhận IP bình thường (hoặc ping được gateway nếu gán tay), truy cập được tài nguyên nội bộ và Internet như các phòng ban khác — `show vlan brief` và `show interfaces trunk` trên SW-EDGE-NEW phản ánh đúng trạng thái đã sửa.

---

**Lưu ý khi viết:** không chèn đáp án (nguyên nhân/lệnh sửa) vào file đề bài — lời giải mẫu để riêng ở `Lab-Tonghop-CCNA-04-Troubleshooting-VLAN-Trunk-Solution.md`.
