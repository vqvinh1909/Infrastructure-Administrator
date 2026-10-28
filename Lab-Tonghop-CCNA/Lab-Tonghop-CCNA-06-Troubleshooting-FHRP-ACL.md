# Lab Tổng Hợp CCNA 06 — Troubleshooting Ticket: HSRP không failover + Partner mất quyền truy cập (Module 05–06)

> Kết hợp trọng tâm: Module 05 (FHRP/HSRP), Module 06 (ACL nâng cao trên biên mạng).
> File: `Lab-Tonghop-CCNA-06-Troubleshooting-FHRP-ACL.md` · Lời giải: `Lab-Tonghop-CCNA-06-Troubleshooting-FHRP-ACL-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Ticket từ Help Desk

> **Ticket #4588 — Ưu tiên: Khẩn cấp**
> 2 sự cố xảy ra cùng đợt bảo trì cuối tuần:
>
> **(a)** Đội hạ tầng tắt thử R-HQ-1 để bảo trì định kỳ, theo thiết kế R-HQ-2 phải nhận vai trò gateway ngay lập tức cho VLAN OFFICE (đã cấu hình HSRP từ trước). Thực tế: **toàn bộ VLAN OFFICE mất mạng hoàn toàn**, chỉ có lại sau khi bật R-HQ-1 lên lại — R-HQ-2 dường như không hề "gánh" thay.
>
> **(b)** Đối tác (Partner) báo không truy cập được website nội bộ (`192.168.100.10`). Đội mạng khẳng định: "Đã thêm dòng permit cho đúng IP/port của Partner vào cuối ACL rồi, không hiểu sao vẫn bị chặn."

## 2. Cấu hình hiện tại — trích đoạn có liên quan

**R-HQ-2 (đứng vai standby cho VLAN OFFICE, VIP thiết kế là `10.10.0.3`):**
```
interface GigabitEthernet0/0.10
 ip address 10.10.0.2 255.255.255.224
 standby 10 ip 10.10.0.30
 standby 10 priority 100
```

**ACL `PARTNER_IN` trên R-HQ-1 (áp `in` trên interface hướng Partner):**
```
ip access-list extended PARTNER_IN
 permit eigrp host 172.16.1.10 host 224.0.0.10
 permit eigrp host 172.16.1.10 host 172.16.1.9
 deny ip any any log
 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 80
 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 443
```

## 3. Yêu cầu chẩn đoán

- [ ] Xác định vì sao R-HQ-2 không thực sự đảm nhận được vai trò gateway thay R-HQ-1 dù `priority 100` đã được cấu hình.
- [ ] Xác định vì sao 2 dòng `permit tcp` mới thêm cho Partner "không có tác dụng" dù cú pháp đúng chuẩn.
- [ ] Đề xuất sửa lỗi tối thiểu cho cả 2 sự cố, giữ nguyên thiết kế (không đổi kiến trúc HSRP hay đổi cách Partner kết nối).

## 4. Ràng buộc

- Không đổi địa chỉ VIP đã thống nhất trong thiết kế (`10.10.0.3`) — sửa đúng chỗ sai, không đổi thiết kế để né lỗi.
- Không xóa và viết lại toàn bộ ACL `PARTNER_IN` từ đầu — chỉ sắp xếp/sửa đúng phần cần thiết.

## 5. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Chẩn đoán HSRP | Phát hiện đúng R-HQ-2 đang đứng cho 1 VIP khác (`10.10.0.30`), không phải VIP thật (`10.10.0.3`) | Đổ lỗi sai cho priority/preempt trong khi bản chất là sai địa chỉ VIP |
| Chẩn đoán ACL | Phát hiện đúng nguyên nhân thứ tự dòng lệnh — `deny ip any any` chặn trước khi tới được dòng `permit` mới | Chỉ kiểm tra nội dung dòng permit mà không xét thứ tự xử lý ACL |
| Cách sửa | Sửa tối thiểu, đúng vị trí, không đổi thiết kế | Viết lại toàn bộ cấu hình liên quan |
| Xác minh | Có `show`/test thực tế xác nhận cả 2 sự cố đã hết | Chỉ khẳng định miệng |

## 6. Gợi ý cách tiếp cận

1. Với sự cố (a): đừng chỉ nhìn `priority` — priority chỉ quyết định *ai thắng khi cả 2 cùng thấy nhau bình thường*; kiểm tra kỹ **địa chỉ IP ảo (VIP)** trên cả 2 router có khớp nhau tuyệt đối hay không.
2. Với sự cố (b): nhớ lại nguyên tắc xử lý ACL của Cisco IOS — xử lý tuần tự từ trên xuống, dừng lại ngay khi gặp dòng đầu tiên khớp điều kiện. Dòng thêm vào **sau** 1 dòng `deny any any` gần như chắc chắn không bao giờ được xét tới.
3. `show access-lists` với bộ đếm match ở từng dòng là cách nhanh nhất xác nhận dòng nào đang thực sự "ăn" traffic.

## 7. Lệnh chẩn đoán cần chạy

- `show standby brief` trên cả R-HQ-1 và R-HQ-2 — so sánh VIP hiển thị trên từng router.
- `show standby` (chi tiết) trên R-HQ-2 — kiểm tra group 10 đang đứng ở địa chỉ nào.
- `show access-lists PARTNER_IN` trên R-HQ-1 — xem thứ tự dòng và bộ đếm match của từng dòng khi có traffic thử từ Partner.
- Tắt thử R-HQ-1 sau khi sửa lỗi (a) — xác nhận PC VLAN OFFICE vẫn ping ra ngoài được liên tục, không mất gói kéo dài.

## 8. Kết quả kỳ vọng sau khi sửa

R-HQ-2 thực sự đứng cùng VIP `10.10.0.3` với R-HQ-1 — tắt thử R-HQ-1 không còn làm gián đoạn VLAN OFFICE. ACL `PARTNER_IN` cho phép đúng traffic HTTP/HTTPS của Partner tới `192.168.100.10`, xác nhận bằng bộ đếm match tăng đúng ở dòng permit khi Partner truy cập thử.

---

**Lưu ý khi viết:** không chèn đáp án (lệnh sửa cụ thể) vào file đề bài — lời giải mẫu để riêng ở `Lab-Tonghop-CCNA-06-Troubleshooting-FHRP-ACL-Solution.md`.
