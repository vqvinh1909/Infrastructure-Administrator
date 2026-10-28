# Mini Lab — Module 01: Nền tảng mạng (CCNA)

> Áp dụng sau khi hoàn thành Bài 01–06.
> File: `Mini-Project-Module-01.md` · Lời giải: `Mini-Project-Module-01-Solution.md`
> Môi trường: giấy/bảng tính + Cisco Packet Tracer (kiểm tra kết nối vật lý và ping).

---

## 1. Bối cảnh doanh nghiệp

Công ty **TechStart JSC** (khoảng 45 người, chưa tính thiết bị khách) chuyển sang văn phòng mới 3 tầng. Bạn là kỹ thuật viên mạng được giao thiết kế lại toàn bộ phần nền tảng trước khi đội switching/routing vào lắp đặt tuần sau: chọn đúng loại cáp cho từng đoạn kết nối, và lập bảng địa chỉ IPv4 (VLSM) + IPv6 cho toàn bộ tòa nhà.

Yêu cầu nhân sự theo tầng:

| Tầng | Phòng ban | Số user | Ghi chú |
|---|---|---|---|
| 1 | Sales + Lễ tân | 25 | 25 user nội bộ; Guest Wifi cho khách (~10 thiết bị) tính riêng, không nằm trong 25 user |
| 2 | Kế toán + Nhân sự | 12 | Dữ liệu nhạy cảm, cần mạng riêng |
| 3 | IT + Server Room | 8 | Đặt Router biên và toàn bộ switch trung tâm |

Khoảng cách cáp thực tế đo được:
- Server Room (tầng 3) → Switch tầng 2: 55m, đi trong ống gen tòa nhà.
- Server Room (tầng 3) → Switch tầng 1: 95m, đi qua khu vực có nhiều động cơ thang máy/nhiễu điện.
- Tòa nhà chính → Tòa nhà phụ (kho, cách 300m) cần 1 văn phòng nhỏ 3 máy để theo dõi kho — không có đường điện đi kèm dễ dàng, cần cân nhắc chống sét lan truyền.

## 2. Sơ đồ mạng (đề bài — chưa gán loại cáp/IP)

```
[ Server Room - Tang 3 (IT) ]
        |
        |--(A) 55m, di trong ong gen, khong nhieu----> [ Switch Tang 2 - Ke toan/Nhan su ]
        |
        |--(B) 95m, gan dong co thang may (nhieu dien)-> [ Switch Tang 1 - Sales/Le tan ]

Toa nha chinh <====== (C) 300m, khong co duong dien di kem ======> Toa nha kho (3 may)
```

## 3. Bảng cần điền

**a) Chọn loại phương tiện truyền dẫn** (điền cột "Loại cáp/media đề xuất" + lý do, dựa trên Bài 03):

| Đoạn | Khoảng cách | Đặc điểm môi trường | Loại cáp/media đề xuất | Lý do |
|---|---|---|---|---|
| (A) Server Room → Tầng 2 | 55m | Trong ống gen, không nhiễu | | |
| (B) Server Room → Tầng 1 | 95m | Gần động cơ thang máy (nhiễu điện) | | |
| (C) Tòa chính → Tòa kho | 300m | Ngoài trời, không có đường điện đi kèm | | |

**b) VLSM cho khối `192.168.10.0/24`** — cấp phát vừa đủ cho từng nhóm (không lãng phí quá nhiều địa chỉ), còn dư ít nhất 1 khối cho mở rộng:

| Subnet | Số host cần | Địa chỉ mạng | Subnet mask (/CIDR) | Địa chỉ dùng được (First–Last) | Broadcast |
|---|---|---|---|---|---|
| Sales + Lễ tân (25 user) | 25 | | | | |
| Guest Wifi (10 thiết bị) | 10 | | | | |
| Kế toán + Nhân sự (12 user) | 12 | | | | |
| IT + Server (8 user) | 8 | | | | |
| Link WAN Tòa chính–Tòa kho (point-to-point) | 2 | | | | |
| Dự phòng mở rộng | — | | | | |

**c) IPv6** — cấp 1 khối `/48` giả định `2001:db8:aabb::/48` cho toàn công ty, chia mỗi tầng 1 subnet `/64` (Bài 06):

| Tầng/khu vực | Subnet IPv6 (/64) |
|---|---|
| Tầng 1 (Sales/Guest) | |
| Tầng 2 (Kế toán/NS) | |
| Tầng 3 (IT/Server) | |

## 4. Yêu cầu kỹ thuật

- [ ] Mỗi đoạn cáp (A, B, C) có 1 loại media cụ thể kèm lý do liên hệ đúng đặc tính vật lý đã học ở Bài 03 (suy hao, nhiễu điện từ, khoảng cách tối đa của từng chuẩn).
- [ ] Bảng VLSM không có subnet nào chồng lấn địa chỉ với subnet khác.
- [ ] Mỗi subnet VLSM có số host dùng được ≥ số host cần, nhưng không lớn hơn quá 2 lần nhu cầu thực tế (trừ khối dự phòng).
- [ ] Subnet mask của link point-to-point WAN dùng đúng loại tối ưu cho 2 host.
- [ ] Bảng IPv6 dùng đúng ranh giới `/64` cho mỗi subnet, không tính sai từ khối `/48`.

## 5. Ràng buộc

- Khối IPv4 công ty được cấp chỉ có `192.168.10.0/24` — không được xin thêm khối khác.
- Đoạn (C) không có sẵn đường điện đi kèm cáp — phải tính đến khả năng chống sét lan truyền/thiết bị nguồn.
- Đoạn (B) nằm gần nguồn nhiễu điện mạnh — không được chọn loại cáp dễ bị suy giảm tín hiệu do nhiễu điện từ (EMI).

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Chọn media | Đúng loại cho cả 3 đoạn, lý do bám đúng đặc tính vật lý (suy hao/nhiễu/khoảng cách) | Chọn cáp đồng cho đoạn (B) hoặc (C) mà không giải thích được rủi ro |
| VLSM | Không chồng lấn, đúng số host tối thiểu, còn dư khối cho mở rộng | Có chồng lấn địa chỉ, hoặc cấp dư quá nhiều gây lãng phí |
| Link WAN | Dùng đúng mask tối ưu cho point-to-point | Dùng /24 hoặc mask không phù hợp cho 2 host |
| IPv6 | Chia đúng ranh giới /64, không tính nhầm từ /48 | Subnet sai ranh giới hoặc trùng nhau |

## 7. Gợi ý triển khai

1. Với phần chọn cáp, liệt kê lại bảng khoảng cách tối đa của UTP/STP/cáp quang (multimode/singlemode) đã học ở Bài 03 trước khi điền — đừng chọn theo cảm tính.
2. Với VLSM, sắp xếp nhu cầu từ lớn đến nhỏ trước khi cấp phát block, để tránh bị "vỡ" khối khi tới nhu cầu nhỏ hơn.
3. Với IPv6, nhớ lại quy tắc rút gọn địa chỉ (Bài 06) khi viết subnet.

## 8. Lệnh verify cần chạy

- Trong Packet Tracer: dựng lại đúng sơ đồ vật lý, gán IP đã tính, dùng `ping` giữa 2 host bất kỳ trong cùng subnet và khác subnet (kỳ vọng: cùng subnet ping được, khác subnet chưa ping được vì chưa có routing — đúng ở giai đoạn Module 01, sẽ xử lý ở Module 03).
- Tự tính lại bằng tay ít nhất 2 subnet trong bảng VLSM để đối chiếu, không chỉ dùng subnet calculator.

## 9. Kết quả kỳ vọng

Có 1 bảng chọn media đầy đủ lý do kỹ thuật cho cả 3 đoạn cáp, 1 bảng VLSM không chồng lấn và tối ưu cho khối `192.168.10.0/24`, và 1 bảng phân subnet IPv6 `/64` đúng ranh giới — đủ để đội switching/routing dùng ngay ở Module 02–03 mà không cần tính lại.

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (loại cáp cụ thể, bảng VLSM đã tính) vào file đề bài — người học tự điền dựa trên kiến thức Bài 01–06.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-01-Solution.md`.
- Đề bài buộc dùng kiến thức từ ít nhất 3 bài khác nhau: Bài 03 (phương tiện truyền dẫn), Bài 05 (VLSM/IPv4), Bài 06 (IPv6).
