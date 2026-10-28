# Mini Lab — Module 05: Remote Access và Wireless (CCNA)

> Áp dụng sau khi hoàn thành Bài 21–25.
> File: `Mini-Project-Module-05.md` · Lời giải: `Mini-Project-Module-05-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Bối cảnh doanh nghiệp

TechStart JSC vừa ký hợp đồng với 1 khách hàng lớn — hệ thống **không được phép có điểm chết** (single point of failure) ở khu vực Sales/IT, và nhân sự bắt đầu làm việc từ xa nên cần VPN vào mạng công ty. Song song đó, văn phòng chuyển sang mô hình Wifi cho toàn bộ nhân viên thay vì cắm dây, quản lý tập trung qua 1 Wireless Controller.

Sếp giao 4 việc: (1) router chính không được là điểm chết duy nhất cho VLAN Sales/IT, (2) đường Internet cũng cần có đường dự phòng, (3) dựng VPN cho nhân viên làm việc từ xa truy cập an toàn vào mạng nội bộ, (4) triển khai Wifi qua WLC cho toàn văn phòng — nhân viên và khách phải tách biệt SSID.

## 2. Sơ đồ & thông tin bổ sung

```
[ISP-A] (duong chinh)          [ISP-B] (duong du phong)
        \                              /
         \                            /
              [ R-HQ-1 ]  (HSRP Active cho VLAN 10, 40)
                    |
              [ SW-DIST ]  (tu Module 02)
                    |
              [ R-HQ-2 ]  (HSRP Standby cho VLAN 10, 40)

  SW-DIST con noi xuong SW-ACC1/ACC2 (Module 02) va them:
  WLC-1 (Wireless Controller) quan ly AP-Floor1, AP-Floor2
```

## 3. Bảng cần điền

**a) HSRP Plan** (cho VLAN 10 – Sales, VLAN 40 – IT/Server; các VLAN khác không yêu cầu trong bài này):

| VLAN | IP R-HQ-1 (Active) | IP R-HQ-2 (Standby) | Virtual IP (HSRP, dùng làm default-gateway) |
|---|---|---|---|
| 10 (Sales) | | | |
| 40 (IT/Server) | | | |

**b) WAN Failover Plan:**

| Link | Vai trò | Cơ chế phát hiện lỗi đề xuất |
|---|---|---|
| R-HQ-1 ↔ ISP-A | Chính | |
| R-HQ-1 ↔ ISP-B | Dự phòng | |

**c) WLAN Plan (trên WLC):**

| SSID | Ánh xạ VLAN | Bảo mật đề xuất | Đối tượng dùng |
|---|---|---|---|
| TECHSTART-CORP | VLAN 10 | | Nhân viên |
| TECHSTART-GUEST | VLAN 20 | | Khách |

## 4. Yêu cầu kỹ thuật

- [ ] VLAN 10 và VLAN 40 có 2 gateway vật lý (R-HQ-1, R-HQ-2) hoạt động dạng active/standby — khi R-HQ-1 down, máy trạm vẫn ping ra ngoài được mà **không cần đổi gateway thủ công**.
- [ ] R-HQ-1 dùng link ISP-A làm đường ra Internet chính; khi ISP-A mất kết nối, tự động chuyển sang ISP-B mà không cần can thiệp tay.
- [ ] Nhân viên làm việc từ xa (giả lập 1 PC ngoài mạng công ty) kết nối VPN vào được mạng nội bộ HQ, sau khi kết nối phải truy cập được tài nguyên nội bộ (ví dụ SRV-WEB ở Module 04) như đang ngồi trong công ty.
- [ ] WLC quản lý tối thiểu 2 AP (mỗi tầng 1 cái), phát 2 SSID riêng biệt ánh xạ đúng VLAN 10 (nhân viên) và VLAN 20 (khách) — thiết bị kết nối SSID nào nhận đúng dải IP của VLAN đó.

## 5. Ràng buộc

- Không dùng 2 địa chỉ IP vật lý của R-HQ-1/R-HQ-2 làm default-gateway cho máy trạm — bắt buộc dùng địa chỉ ảo (virtual IP) của cơ chế redundancy đã học.
- WAN failover phải tự động phát hiện mất kết nối ISP-A (không phải kiểu "thấy mất mạng thì admin vào đổi route tay").
- SSID khách (`TECHSTART-GUEST`) không được dùng chung pre-shared key với SSID nhân viên, và không được lộ vào VLAN nội bộ khác ngoài VLAN 20.
- Do có HSRP thay đổi default-gateway của VLAN 10/40, cần rà soát lại cấu hình DHCP đã làm ở Module 04 (default-router) để không bị lệch với gateway mới.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| HSRP | Failover mượt, máy trạm không mất gateway khi R-HQ-1 down, priority/preempt hợp lý | Cả 2 router cùng active (chưa cấu hình priority), hoặc không preempt lại đúng khi Active hồi phục |
| WAN Failover | Tự động chuyển ISP-B khi ISP-A mất, tự quay lại ISP-A khi hồi phục | Phải cấu hình tay khi đổi ISP, hoặc không có cơ chế phát hiện lỗi |
| VPN | Remote PC kết nối và truy cập được tài nguyên nội bộ qua đường mã hóa | Kết nối được nhưng không mã hóa, hoặc không truy cập được tài nguyên nội bộ |
| WLAN/WLC | 2 SSID tách biệt đúng VLAN, đúng bảo mật, AP join WLC thành công | SSID lẫn VLAN, dùng chung password, hoặc AP không join được WLC |
| DHCP rà soát lại | Default-router trong DHCP pool khớp đúng virtual IP mới | Vẫn để default-router là IP vật lý cũ của R-HQ-1 |

## 7. Gợi ý triển khai

1. Cấu hình HSRP trước (Bài 21) — dựng xong mới điều chỉnh DHCP default-router, tránh phải sửa đi sửa lại 2 lần.
2. WAN failover (Bài 23) nên dùng cơ chế theo dõi trạng thái đường truyền (tracking object) kết hợp floating static route — không chỉ đơn giản "route với AD thấp hơn" mà không theo dõi gì cả.
3. VPN Remote Access (Bài 22) cấu hình xong nên test bằng cách ping/truy cập thử tài nguyên nội bộ từ máy remote, không chỉ kiểm tra "kết nối thành công" trên giao diện client.
4. Với WLC (Bài 24-25), tạo Interface/VLAN mapping trên WLC trước khi tạo WLAN — thứ tự ngược lại sẽ khiến WLAN không thấy interface để chọn.

## 8. Lệnh verify cần chạy

- `show standby brief` trên cả R-HQ-1 và R-HQ-2 — xác nhận đúng vai trò Active/Standby và priority.
- Tắt thử interface WAN của R-HQ-1 (giả lập sự cố) — kiểm tra máy trạm vẫn ping ra ngoài được và `show ip route` thấy route chuyển sang ISP-B.
- `show track` (hoặc lệnh tương ứng) — xác nhận object tracking đang giám sát đúng đường ISP-A.
- Từ remote PC: kết nối VPN, sau đó `ping`/truy cập SRV-WEB nội bộ — thành công.
- Trên WLC: `show wlan summary`, `show ap join stats` — xác nhận đủ 2 WLAN và AP đã join.

## 9. Kết quả kỳ vọng

VLAN Sales/IT có gateway dự phòng qua HSRP, đường Internet có ISP dự phòng tự động chuyển khi ISP chính lỗi, nhân viên làm việc từ xa vào được mạng nội bộ qua VPN mã hóa, và toàn văn phòng phủ Wifi qua WLC với 2 SSID tách biệt đúng VLAN nhân viên/khách — hệ thống không còn điểm chết đơn lẻ ở các khu vực trọng yếu.

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (địa chỉ HSRP cụ thể, cấu hình VPN/WLC) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-05-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 21 (FHRP/HSRP), Bài 22 (Remote Access VPN), Bài 23 (Network Load Balancing/Failover), Bài 24-25 (Wireless LAN, Cisco WLC).
