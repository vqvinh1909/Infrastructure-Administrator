# Lab Tổng Hợp CCNA 05 — Troubleshooting Ticket: OSPF lên nhưng chi nhánh không ra được Internet (Module 03–04)

> Kết hợp trọng tâm: Module 03 (Routing/OSPF), Module 04 (NAT).
> File: `Lab-Tonghop-CCNA-05-Troubleshooting-OSPF-NAT.md` · Lời giải: `Lab-Tonghop-CCNA-05-Troubleshooting-OSPF-NAT-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Ticket từ Help Desk

> **Ticket #4502 — Ưu tiên: Cao**
> Store B báo toàn bộ nhân viên không mở được bất kỳ website nào từ sáng nay. Đã kiểm tra: máy tính vẫn ping được server nội bộ ở HQ bình thường (địa chỉ private), dây mạng/router không có gì bất thường, `show ip ospf neighbor` trên router Store B thấy neighbor với HQ ở trạng thái **Full** — tức là 2 router "thấy nhau" bình thường. Nhưng `ping 8.8.8.8` từ Store B thì timeout.

## 2. Sơ đồ mạng liên quan

```
   [ISP] ---- R-HQ ---- (WAN OSPF) ---- R-STOREB ---- LAN Store B (nhan vien)
```

## 3. Cấu hình hiện tại — trích đoạn có liên quan

**R-HQ:**
```
router ospf 1
 network 10.10.0.0 0.0.0.31 area 0
 network 10.10.0.72 0.0.0.3 area 0
 network 10.10.0.76 0.0.0.3 area 0

ip route 0.0.0.0 0.0.0.0 10.10.0.82

access-list 1 permit 10.10.0.0 0.0.0.255
ip nat inside source list 1 interface Serial0/0/1 overload
```

**R-STOREB** (LAN Store B: `10.10.2.0/28`):
```
router ospf 1
 network 10.10.2.0 0.0.0.15 area 0
 network 10.10.0.76 0.0.0.3 area 0
```

## 4. Yêu cầu chẩn đoán

- [ ] Xác định vì sao Store B **không học được đường ra Internet** dù OSPF neighbor đã Full.
- [ ] Giả sử vấn đề trên đã được xử lý — xác định thêm 1 lỗi khác khiến traffic của Store B **vẫn không ra được Internet** dù đã có route (gợi ý: kiểm tra kỹ điều gì xảy ra tại chính R-HQ khi traffic của Store B đi qua).
- [ ] Đề xuất cách sửa tối thiểu cho cả 2 lỗi, không đổi kiến trúc định tuyến/NAT hiện có.

## 5. Ràng buộc

- Không thêm static route thủ công trên R-STOREB trỏ thẳng ra Internet — bài toán yêu cầu Store B phải học route qua OSPF đúng như thiết kế ban đầu.
- Không đổi cách NAT đang dùng (vẫn dùng access-list + overload) — chỉ sửa đúng phạm vi bị thiếu.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Lỗi 1 (OSPF default route) | Xác định đúng nguyên nhân Store B không có route `0.0.0.0/0`, sửa đúng lệnh | Đoán sai nguyên nhân (ví dụ đổ lỗi cho neighbor chưa Full dù ticket đã nói rõ Full) |
| Lỗi 2 (NAT) | Nhận ra access-list NAT không bao phủ dải LAN Store B | Không kiểm tra NAT, chỉ dừng ở lỗi 1 |
| Cách sửa | Tối thiểu, không đổi kiến trúc | Đổi sang static route hoặc đổi cơ chế NAT khác hẳn |
| Xác minh | Có `show`/`ping` cụ thể chứng minh Store B ra Internet được | Chỉ khẳng định miệng |

## 7. Gợi ý cách tiếp cận

1. OSPF neighbor Full chỉ chứng minh 2 router trao đổi được thông tin định tuyến — không chứng minh route cụ thể nào đó (như default route) đã thực sự có mặt trong bảng định tuyến của Store B. Kiểm tra thẳng bảng định tuyến trước khi kết luận.
2. Nếu route đã có nhưng vẫn không ra được Internet, vấn đề nằm ở bước xử lý traffic tại điểm ra biên (R-HQ) — nhớ lại cơ chế NAT overload chỉ dịch địa chỉ cho các dải nằm trong access-list được tham chiếu.
3. So sánh cẩn thận dải IP trong access-list NAT với dải IP thực tế của từng site — đừng chỉ đọc lướt "có access-list rồi là chắc đủ".

## 8. Lệnh chẩn đoán cần chạy

- `show ip ospf neighbor` — xác nhận lại trạng thái Full (đã biết, nhưng vẫn nên tự kiểm chứng).
- `show ip route` trên R-STOREB — tìm dòng `0.0.0.0/0`, xem có tồn tại và học qua nguồn nào.
- `show run | section router ospf` trên R-HQ — kiểm tra có `default-information originate` hay chưa.
- `show access-lists 1` trên R-HQ — kiểm tra dải nào đang thực sự được NAT.
- `show ip nat translations` khi có traffic thử từ Store B — xem traffic có được dịch địa chỉ hay không.

## 9. Kết quả kỳ vọng sau khi sửa

R-STOREB có route `0.0.0.0/0` học qua OSPF từ R-HQ, và traffic từ LAN Store B (`10.10.2.0/28`) được NAT overload đúng cách khi ra `Serial0/0/1` — nhân viên Store B truy cập Internet bình thường, xác minh bằng `ping 8.8.8.8` và `show ip nat translations` có bản ghi tương ứng.

---

**Lưu ý khi viết:** không chèn đáp án (lệnh sửa cụ thể) vào file đề bài — lời giải mẫu để riêng ở `Lab-Tonghop-CCNA-05-Troubleshooting-OSPF-NAT-Solution.md`.
