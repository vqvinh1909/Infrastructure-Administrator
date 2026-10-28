# Mini Lab — Module 04: Dịch vụ và bảo mật cơ bản (CCNA)

> Áp dụng sau khi hoàn thành Bài 16–20.
> File: `Mini-Project-Module-04.md` · Lời giải: `Mini-Project-Module-04-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Bối cảnh doanh nghiệp

Hạ tầng routing của TechStart JSC (Module 03) đã thông suốt. Giờ IT Manager yêu cầu hoàn thiện 4 việc trước khi đưa vào vận hành chính thức: (1) máy trạm phải tự nhận IP thay vì gán tay từng máy, (2) Guest Wifi phải ra được Internet nhưng **tuyệt đối không** chạm được vào mạng nội bộ, (3) Partner chỉ được phép truy cập đúng 1 web server nội bộ qua HTTP/HTTPS — không được đụng gì khác, (4) toàn bộ router phải đồng bộ giờ và gửi log về 1 nơi tập trung để khi có sự cố còn biết "chuyện gì xảy ra lúc mấy giờ".

## 2. Sơ đồ & thông tin bổ sung

Kế thừa toàn bộ topology Module 03 (R-HQ, R-BRANCH, R-DR, R-PARTNER, R-ISP). Bổ sung 2 máy chủ mới trong VLAN 40 (IT_SERVER, `192.168.10.64/28`):

| Máy chủ | IP đề xuất (trong dải VLAN 40) | Vai trò |
|---|---|---|
| SRV-DHCP | 192.168.10.67 | Cấp DHCP tập trung cho VLAN 30 (Kế toán/NS) |
| SRV-WEB | 192.168.10.70 | Web server nội bộ, Partner cần truy cập HTTP/HTTPS |
| SRV-SYSLOG | 192.168.10.66 | Nhận log tập trung từ toàn bộ router |

## 3. Bảng cần điền

**a) DHCP Pool Plan** (điền dải cấp phát + địa chỉ loại trừ cho từng VLAN):

| VLAN | Dải cấp DHCP | Địa chỉ loại trừ (exclude) | Cấp bởi |
|---|---|---|---|
| VLAN 10 (Sales) | | | Router R-HQ (DHCP cục bộ) |
| VLAN 20 (Guest) | | | Router R-HQ (DHCP cục bộ) |
| VLAN 30 (Kế toán/NS) | | | SRV-DHCP (qua DHCP relay) |

**b) ACL Plan:**

| Vị trí áp ACL | Loại ACL | Mục đích |
|---|---|---|
| Subinterface Gi0/0.20 (Guest) | | Chặn Guest truy cập vào toàn bộ `192.168.10.0/24` khác VLAN của chính nó, vẫn cho ra Internet |
| Serial hướng R-PARTNER trên R-HQ | | Chỉ cho phép Partner truy cập SRV-WEB (80/443), chặn mọi thứ khác vào mạng nội bộ HQ |

## 4. Yêu cầu kỹ thuật

- [ ] VLAN 10 và VLAN 20 nhận IP tự động từ chính R-HQ (DHCP server cục bộ trên router).
- [ ] VLAN 30 nhận IP tự động từ SRV-DHCP (không nằm trên router) — R-HQ phải relay DHCP request sang đúng server này.
- [ ] Guest Wifi (VLAN 20) ra Internet được (qua NAT) nhưng ping/truy cập vào bất kỳ VLAN nội bộ nào khác đều bị chặn.
- [ ] Toàn bộ mạng nội bộ ra Internet qua NAT overload (PAT) dùng địa chỉ public của R-HQ.
- [ ] SRV-WEB được NAT tĩnh ra 1 địa chỉ public riêng để bên ngoài (Partner) truy cập được đúng cổng 80/443, không public toàn bộ server.
- [ ] Partner chỉ truy cập được SRV-WEB đúng 2 cổng trên — mọi traffic khác từ Partner vào mạng nội bộ HQ đều bị chặn.
- [ ] Toàn bộ router đồng bộ giờ theo 1 nguồn NTP chung (R-HQ làm master hoặc đồng bộ từ nguồn ngoài, các router khác làm client).
- [ ] Toàn bộ router gửi log sự kiện quan trọng (thay đổi cấu hình, interface up/down) về SRV-SYSLOG.

## 5. Ràng buộc

- ACL chặn Guest phải đặt đúng chiều và đúng vị trí để không vô tình chặn luôn cả traffic ra Internet của chính Guest.
- Không NAT tĩnh toàn bộ SRV-WEB (không expose SSH/RDP hay các cổng quản trị khác ra ngoài) — chỉ đúng 80/443.
- DHCP pool không được cấp trùng vào các địa chỉ đã dùng cho gateway hoặc server tĩnh (SRV-DHCP, SRV-WEB, SRV-SYSLOG).
- Giờ hệ thống phải đồng nhất — chênh lệch giờ giữa các router sẽ làm log ở SRV-SYSLOG không thể đối chiếu được theo đúng thứ tự thời gian.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| DHCP | VLAN 10/20/30 đều tự nhận IP đúng dải, đúng exclude, VLAN 30 nhận qua relay thành công | Máy trạm không nhận được IP, hoặc nhận nhầm dải |
| ACL Guest | Guest ra Internet được, không vào được VLAN nội bộ khác | Guest bị chặn luôn cả Internet, hoặc vẫn vào được VLAN khác |
| NAT | PAT hoạt động cho toàn mạng nội bộ, static NAT đúng cho SRV-WEB | Thiếu NAT cho 1 nhánh, hoặc static NAT lộ nguyên IP nội bộ |
| ACL Partner | Partner chỉ vào được 80/443 của SRV-WEB, mọi thứ khác bị chặn có log | ACL quá lỏng (Partner vào được chỗ khác) hoặc quá chặt (chặn luôn 80/443) |
| NTP/Syslog | Toàn bộ router đồng bộ giờ, log tập trung về đúng SRV-SYSLOG | Thiếu router chưa đồng bộ, hoặc log không gửi đi |

## 7. Gợi ý triển khai

1. Cấu hình DHCP trước NAT/ACL — máy trạm cần có IP mới kiểm tra được kết nối thực tế ở các bước sau.
2. Với VLAN 30, xem lại khái niệm DHCP Relay (`ip helper-address`) ở Bài 19 — đặt đúng trên subinterface Gi0/0.30, không phải trên SRV-DHCP.
3. Viết ACL Guest theo nguyên tắc "deny cụ thể trước, permit any sau" và nhớ dòng `permit ip any any` cuối cùng nếu không sẽ chặn luôn cả Internet (deny ngầm định).
4. Với ACL Partner, dùng Extended ACL vì cần lọc theo cổng dịch vụ (80/443), Standard ACL không làm được việc này.
5. Cấu hình NTP/Syslog nên làm cuối cùng — không ảnh hưởng kết nối, dễ kiểm chứng bằng `show clock` và log trên SRV-SYSLOG.

## 8. Lệnh verify cần chạy

- `show ip dhcp binding` trên R-HQ (còn SRV-DHCP là server GUI của Packet Tracer, không có lệnh này — xem bảng lease trong tab DHCP của server) — xác nhận máy trạm đã nhận IP đúng dải.
- `show access-lists` — đếm số packet match từng dòng ACL để xác nhận rule đang thực sự chặn/cho phép đúng.
- `show ip nat translations` — xác nhận PAT và static NAT hoạt động khi có traffic thật.
- `show ntp status` / `show ntp associations` trên các router client — xác nhận đã đồng bộ (`synced`).
- Test thực tế: từ Guest ping vào VLAN Sales (phải fail), từ Guest ping ra ngoài Internet (phải thành công), từ Partner truy cập `http://<IP public SRV-WEB>` (phải thành công), từ Partner ping vào IP nội bộ HQ khác (phải fail).

## 9. Kết quả kỳ vọng

Toàn bộ VLAN nhận IP tự động đúng nguồn (local hoặc relay), Guest Wifi cách ly hoàn toàn khỏi mạng nội bộ nhưng vẫn ra Internet được, mạng nội bộ ra ngoài qua NAT overload, Partner chỉ chạm được đúng SRV-WEB ở cổng 80/443, và toàn bộ router đồng bộ giờ + gửi log về SRV-SYSLOG — đủ điều kiện đưa hệ thống vào vận hành thật.

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (dải DHCP cụ thể, cấu hình ACL/NAT) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-04-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 16-17 (ACL Standard/Extended), Bài 18 (NTP/Syslog), Bài 19 (DHCP + Relay), Bài 20 (NAT static/overload).
