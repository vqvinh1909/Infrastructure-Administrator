---
title: README - Quản trị Hạ tầng mạng (Infrastructure Administrator)
type: Overview
last_review: 2026-07-21
---

# Quản trị Hạ tầng mạng (Infrastructure Administrator)

Khóa học 30 bài, chia làm 7 module, xây dựng năng lực Sysadmin/Network Administrator từ nền tảng tới thực chiến — đủ để tự triển khai hạ tầng mạng doanh nghiệp, tự xử lý sự cố, và tự tin vượt qua phỏng vấn Sysadmin Junior/Mid-level.

## Cách dùng tài liệu này

Mỗi bài học (`Bai-XX_*.md`) đều theo cùng 1 cấu trúc:

- **Giải thích khái niệm** — đi từ "tại sao" trước khi tới "làm như thế nào", liên hệ với môi trường doanh nghiệp thực tế.
- **Sơ đồ ASCII** — minh họa kiến trúc/luồng dữ liệu, có thể xem trực tiếp trong bất kỳ trình soạn thảo Markdown nào.
- **Case study thực tế** — sự cố/case study đã xảy ra ngoài đời, có trích dẫn nguồn, giúp gắn kiến thức với hậu quả thực tế thay vì chỉ lý thuyết suông.
- **Cấu hình mẫu** — lệnh CLI Cisco cụ thể cho từng tình huống.
- **Verify & Troubleshoot** — bảng triệu chứng/nguyên nhân/cách xử lý, tư duy chẩn đoán sự cố.
- **Lab thực hành** — bài tập cụ thể để tự làm trên Packet Tracer/GNS3.
- **Ngộ nhận thường gặp** — các hiểu lầm phổ biến và lý do vì sao sai.
- **Câu hỏi phỏng vấn nền tảng** — kèm đáp án chuẩn, để tự luyện trước khi phỏng vấn.
- **Tài liệu tham khảo** — nguồn đã xác minh (Cisco, tài liệu kỹ thuật uy tín).

Các callout `[!question]` trong mỗi bài là câu hỏi tư duy sâu (không chỉ hỏi "là gì" mà hỏi "vì sao") — nên tự trả lời trước khi đọc phần giải thích, để kiểm tra mức độ hiểu thực sự thay vì chỉ đọc lướt.

## Cấu trúc khóa học

| Module | Chủ đề | Bài học |
|---|---|---|
| **1. Nền tảng mạng** | Khái niệm cơ bản, mô hình OSI/TCP-IP, phương tiện truyền dẫn, thiết bị, địa chỉ IPv4/IPv6 | [[Module-01_Nen-tang-mang/Bai-01_Gioi-thieu-mang-may-tinh\|Bài 1]] · [[Module-01_Nen-tang-mang/Bai-02_Mo-hinh-OSI-TCPIP\|Bài 2]] · [[Module-01_Nen-tang-mang/Bai-03_Phuong-tien-truyen-dan-mang\|Bài 3]] · [[Module-01_Nen-tang-mang/Bai-04_Thiet-bi-ket-noi-mang\|Bài 4]] · [[Module-01_Nen-tang-mang/Bai-05_Dia-chi-IPv4\|Bài 5]] · [[Module-01_Nen-tang-mang/Bai-06_Dia-chi-IPv6\|Bài 6]] |
| **2. Switching** | VLAN, Trunking/DTP/VTP, STP, EtherChannel | [[Module-02_Switching/Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco\|Bài 7]] · [[Module-02_Switching/Bai-08_Cau-hinh-va-quan-ly-VLAN\|Bài 8]] · [[Module-02_Switching/Bai-09_Trunking-DTP-VTP\|Bài 9]] · [[Module-02_Switching/Bai-10_STP-va-EtherChannel\|Bài 10]] |
| **3. Routing** | Định tuyến cơ bản, Inter-VLAN Routing, Static Route, OSPF, EIGRP | [[Module-03_Routing/Bai-11_Gioi-thieu-ve-Dinh-tuyen\|Bài 11]] · [[Module-03_Routing/Bai-12_Inter-VLAN-Routing\|Bài 12]] · [[Module-03_Routing/Bai-13_Static-Route\|Bài 13]] · [[Module-03_Routing/Bai-14_OSPF\|Bài 14]] · [[Module-03_Routing/Bai-15_EIGRP\|Bài 15]] |
| **4. Dịch vụ và bảo mật cơ bản** | ACL (Standard/Extended), NTP/Syslog, DHCP, NAT | [[Module-04_Dich-vu-va-Bao-mat-co-ban/Bai-16_Basic-ACL\|Bài 16]] · [[Module-04_Dich-vu-va-Bao-mat-co-ban/Bai-17_Advanced-ACL\|Bài 17]] · [[Module-04_Dich-vu-va-Bao-mat-co-ban/Bai-18_NTP-va-Syslog\|Bài 18]] · [[Module-04_Dich-vu-va-Bao-mat-co-ban/Bai-19_Dich-vu-DHCP\|Bài 19]] · [[Module-04_Dich-vu-va-Bao-mat-co-ban/Bai-20_NAT\|Bài 20]] |
| **5. Remote Access và Wireless** | FHRP (HSRP/VRRP/GLBP), VPN, Load Balancing, Wireless LAN, Cisco WLC | [[Module-05_Remote-Access-va-Wireless/Bai-21_FHRP\|Bài 21]] · [[Module-05_Remote-Access-va-Wireless/Bai-22_Remote-Access-VPN\|Bài 22]] · [[Module-05_Remote-Access-va-Wireless/Bai-23_Network-Load-Balancing\|Bài 23]] · [[Module-05_Remote-Access-va-Wireless/Bai-24_Wireless-LAN\|Bài 24]] · [[Module-05_Remote-Access-va-Wireless/Bai-25_Cisco-Wireless-Controller\|Bài 25]] |
| **6. Bảo mật nâng cao** | Nguy cơ bảo mật (Spoofing/DoS/MITM/Malware...), cấu hình bảo mật thiết bị (Password/ACL/AAA), Firewall ASA | [[Module-06_Bao-mat-nang-cao/Bai-26_Bao-mat-Lop-2\|Bài 26]] · [[Module-06_Bao-mat-nang-cao/Bai-27_AAA\|Bài 27]] · [[Module-06_Bao-mat-nang-cao/Bai-28_Cisco-ASA-Firewall\|Bài 28]] |
| **7. Automation và Programmability** | Network Automation, SDN, Python/Netmiko, Ansible | [[Module-07_Automation-va-Programmability/Bai-29_Network-Automation-va-Programmability\|Bài 29]] · [[Module-07_Automation-va-Programmability/Bai-30_Ansible-va-Tong-ket\|Bài 30]] |

## Lab tổng hợp thực hành (Mini Project & Capstone)

Mỗi module sau khi học xong `Bai-XX` có 1 **Mini Project** riêng — đề bài tổng hợp kiến thức của cả module thành 1 bài toán doanh nghiệp (công ty **TechStart JSC**, xuyên suốt 7 module), kèm file lời giải mẫu tách riêng:

| Module | Đề bài | Lời giải mẫu |
|---|---|---|
| 1. Nền tảng mạng | [[Module-01_Nen-tang-mang/Mini-Project-Module-01\|Mini-Project-Module-01]] | [[Module-01_Nen-tang-mang/Mini-Project-Module-01-Solution\|Solution]] |
| 2. Switching | [[Module-02_Switching/Mini-Project-Module-02\|Mini-Project-Module-02]] | [[Module-02_Switching/Mini-Project-Module-02-Solution\|Solution]] |
| 3. Routing | [[Module-03_Routing/Mini-Project-Module-03\|Mini-Project-Module-03]] | [[Module-03_Routing/Mini-Project-Module-03-Solution\|Solution]] |
| 4. Dịch vụ và bảo mật cơ bản | [[Module-04_Dich-vu-va-Bao-mat-co-ban/Mini-Project-Module-04\|Mini-Project-Module-04]] | [[Module-04_Dich-vu-va-Bao-mat-co-ban/Mini-Project-Module-04-Solution\|Solution]] |
| 5. Remote Access và Wireless | [[Module-05_Remote-Access-va-Wireless/Mini-Project-Module-05\|Mini-Project-Module-05]] | [[Module-05_Remote-Access-va-Wireless/Mini-Project-Module-05-Solution\|Solution]] |
| 6. Bảo mật nâng cao | [[Module-06_Bao-mat-nang-cao/Mini-Project-Module-06\|Mini-Project-Module-06]] | [[Module-06_Bao-mat-nang-cao/Mini-Project-Module-06-Solution\|Solution]] |
| 7. Automation và Programmability | [[Module-07_Automation-va-Programmability/Mini-Project-Module-07\|Mini-Project-Module-07]] | [[Module-07_Automation-va-Programmability/Mini-Project-Module-07-Solution\|Solution]] |

Sau khi hoàn thành cả 30 bài + 7 Mini Project, thư mục [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-01-Full-GreenMart\|Lab-Tonghop-CCNA/]] có **6 bài lab tổng hợp cuối khóa**:

**Lab Full** — mỗi bài 1 topology duy nhất, danh sách 20 Task nối tiếp phủ đủ cả 7 module (VLSM → Switching → Routing → Dịch vụ cơ bản → HA/VPN/Wireless → Bảo mật nâng cao → Automation), mỗi Task có kết quả kiểm tra cụ thể — đúng format "lab tổng hợp" thi/phỏng vấn CCNA thực tế (tham khảo cấu trúc từ waren.vn, PingLabz CCNA Mega Lab, GitHub dannyhng/ccna-full-config-lab). 3 bài dùng 3 kịch bản/kỹ thuật khác nhau để tránh lặp, luyện lại toàn bộ kiến thức nhiều lần:

| Bài | Kịch bản | Điểm khác biệt kỹ thuật |
|---|---|---|
| [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-01-Full-GreenMart\|CCNA-01]] | GreenMart Retail (bán lẻ) | OSPF, HSRP, dual-ISP failover, ASA+DMZ (web store), 1 SSID |
| [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-02-Full-VietManu\|CCNA-02]] | VietManu Manufacturing (nhà máy) | EIGRP, VRRP, static route (site vệ tinh), cách ly OT/IT (VLAN sản xuất không NAT/DHCP), Port Security shutdown cho thiết bị OT |
| [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-03-Full-SunCare\|CCNA-03]] | SunCare Hospital (bệnh viện) | GLBP, OSPF có xác thực MD5, 2 SSID Wifi, cách ly dữ liệu nhạy cảm (2 VLAN không NAT) |

**Troubleshooting ticket** (cho sẵn topology + cấu hình có lỗi, yêu cầu chẩn đoán và sửa): [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-04-Troubleshooting-VLAN-Trunk\|CCNA-04]] (Module 02), [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-05-Troubleshooting-OSPF-NAT\|CCNA-05]] (Module 03–04), [[Lab-Tonghop-CCNA/Lab-Tonghop-CCNA-06-Troubleshooting-FHRP-ACL\|CCNA-06]] (Module 05–06).

Mỗi bài đều có file `-Solution.md` tách riêng — chỉ mở ra sau khi đã tự làm xong, không đọc trước.

### Quy trình tự học 1 bài (áp dụng chung cho cả 7 Mini Project và 6 Lab tổng hợp)

1. **Đọc đề 1 lần, chưa động vào CLI** — nắm bối cảnh doanh nghiệp, yêu cầu kỹ thuật, và đặc biệt là mục "Ràng buộc" (đây là nơi hay chứa các bẫy dễ bỏ sót).
2. **Điền các bảng còn trống trên giấy/bảng tính trước** (VLSM, VLAN plan, security level...) — đừng vừa nghĩ vừa gõ router, dễ tính sai và phải sửa đi sửa lại.
3. **Triển khai trên Packet Tracer/GNS3**, bám đúng yêu cầu kỹ thuật và rubric — không thêm/bớt phạm vi ngoài đề.
4. **Tự chạy hết mục "Lệnh verify cần chạy" / "Lệnh chẩn đoán cần chạy"** trong đề — xác nhận bằng bằng chứng thật (`show`, `ping`), không dừng lại chỉ vì "có vẻ chạy được".
5. **Tự chấm theo bảng "Tiêu chí đánh giá (Rubric)"** trong đề bài — chấm khách quan theo từng dòng, không tự nhận "làm được rồi" khi chưa đối chiếu rubric.
6. **Mở file `-Solution.md` sau khi đã làm xong (hoặc bí thật sự)** — đọc kỹ mục "Giải thích các điểm quan trọng / dễ sai" ở cuối, đây là phần quan trọng hơn cả việc so khớp câu lệnh giống hay khác.
7. **Nếu sai/thiếu, quay lại đúng `Bai-XX` được ghi ở "Lưu ý khi viết" cuối đề bài** để ôn lại kiến thức gốc, rồi làm lại từ đầu — không chỉ sửa đúng chỗ sai rồi bỏ qua.

Với 3 bài **Lab Full** (CCNA-01/02/03), nên làm theo đúng thứ tự Task 1→20 vì nhiều Task phụ thuộc kết quả Task trước — làm cả 3 bài (không chỉ 1) để luyện lại toàn bộ kiến thức với kịch bản/kỹ thuật khác nhau mỗi lần, tránh học thuộc lòng 1 kịch bản duy nhất.

Riêng 3 bài **Troubleshooting Ticket** (CCNA-04, CCNA-05, CCNA-06): tự đặt giới hạn thời gian chẩn đoán (khoảng 30–45 phút) trước khi mở Solution — mục tiêu là luyện phản xạ xử lý sự cố dưới áp lực thời gian, sát với phỏng vấn thực chiến hơn là "chắc chắn tìm ra bằng mọi giá".

## Yêu cầu môi trường thực hành

- **Cisco Packet Tracer** — dùng cho phần lớn lab trong khóa (VLAN, Routing, ACL, NAT, FHRP, Wireless, WLC...).
- **GNS3 hoặc EVE-NG** — cần cho các lab mà Packet Tracer mô phỏng hạn chế hoặc không hỗ trợ đầy đủ (IPsec VPN chi tiết ở Bài 22, ASA nâng cao ở Bài 28).
- **Python 3 + pip** — cho Module 7 (`pip install netmiko ansible`).
- Kiến thức nền: không yêu cầu kinh nghiệm làm việc trước đó — mỗi bài đều giải thích từ khái niệm cơ bản, phù hợp người mới bắt đầu học Sysadmin.

## Lộ trình học đề xuất

Học tuần tự theo đúng thứ tự Module 1 → 7 — mỗi module xây dựng trên kiến thức của module trước (ví dụ: hiểu VLAN ở Module 2 là điều kiện bắt buộc để hiểu Inter-VLAN Routing ở Module 3; hiểu NAT/ACL ở Module 4 là nền tảng để hiểu Firewall ASA ở Module 6). Sau mỗi module, nên tự trả lời toàn bộ câu hỏi phỏng vấn ở cuối từng bài, **tự làm Mini Project của module đó trước khi mở file Solution**, rồi mới chuyển sang module tiếp theo.

Sau khi hoàn thành cả 30 bài + 7 Mini Project, làm tiếp 6 bài Lab tổng hợp CCNA cuối khóa (`Lab-Tonghop-CCNA/`) — đây là bước luyện tư duy tổng hợp nhiều module cùng lúc, gần với thực chiến và phỏng vấn nhất trong toàn khóa. Sau đó xem gợi ý hướng đi tiếp theo ở mục 12 của [[Module-07_Automation-va-Programmability/Bai-30_Ansible-va-Tong-ket|Bài 30]].
