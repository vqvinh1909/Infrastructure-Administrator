---
title: Bài 18 - NTP và Syslog
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — NTP và Syslog là 2 dịch vụ nền tảng bắt buộc phải có trên mọi hạ tầng doanh nghiệp trước khi nói đến giám sát hay điều tra sự cố"
tags: [networking, cisco, ntp, syslog, logging, monitoring, sysadmin]
aliases: [NTP, Syslog, Network Time Protocol, Log Server]
related: ["[[Bai-17_Advanced-ACL]]", "[[Bai-19_Dich-vu-DHCP]]"]
last_review: 2026-07-21
---

# Bài 18: NTP và Syslog

> [!important] Tại sao cần file này
> Mọi kỹ năng troubleshoot đã học từ Bài 7 đến Bài 17 đều dựa trên 1 giả định ngầm: **log và thời gian trên thiết bị là chính xác**. Nếu đồng hồ router lệch nhau vài phút giữa các thiết bị, việc đối chiếu log để tìm ra "sự kiện A trên switch xảy ra trước hay sau sự kiện B trên router" trở nên vô nghĩa — đây chính xác là lý do NTP (đồng bộ thời gian) và Syslog (thu thập log tập trung) luôn được cấu hình **cùng nhau** và luôn nằm trong nhóm việc đầu tiên khi triển khai bất kỳ thiết bị mới nào.

## 1. NTP và Syslog là gì

- **NTP (Network Time Protocol)**: giao thức đồng bộ thời gian giữa các thiết bị trong mạng với 1 nguồn thời gian chuẩn, đảm bảo mọi thiết bị (router, switch, server, firewall...) có cùng 1 đồng hồ thống nhất.
- **Syslog**: giao thức chuẩn để thiết bị mạng gửi log (nhật ký sự kiện) tới 1 máy chủ tập trung (Syslog Server), thay vì log chỉ nằm rời rạc trên từng thiết bị.

> [!question] Vì sao NTP luôn được xem là điều kiện tiên quyết trước khi triển khai Syslog tập trung?
> Vì giá trị lớn nhất của Syslog tập trung là khả năng **đối chiếu trình tự sự kiện giữa nhiều thiết bị khác nhau** khi điều tra sự cố (ví dụ: switch ghi nhận cổng down lúc nào, router ghi nhận route biến mất lúc nào — cái nào xảy ra trước?). Nếu đồng hồ các thiết bị không đồng bộ, timestamp trong log sẽ sai lệch, khiến việc dựng lại đúng trình tự sự kiện trở nên không đáng tin cậy hoặc thậm chí gây kết luận sai hoàn toàn về nguyên nhân gốc rễ sự cố.

## 2. Thiết lập ngày giờ, múi giờ, và NTP Server

**Đặt giờ/múi giờ thủ công** (dùng khi chưa có NTP hoặc để kiểm tra ban đầu):

```
Router# clock set 14:30:00 21 July 2026
Router(config)# clock timezone ICT 7
```

`clock timezone ICT 7` khai báo múi giờ Việt Nam (Indochina Time, UTC+7) — quan trọng vì NTP luôn truyền tải thời gian theo chuẩn **UTC**, thiết bị tự tính lại giờ địa phương dựa trên độ lệch múi giờ đã khai báo.

**Cấu hình đồng bộ qua NTP Server:**

```
Router(config)# ntp server 10.10.10.100
```

```
NTP Stratum hierarchy:
+--------------------+   +--------------------+   +--------------------+   +----------------------+
|  GPS/Atomic Clock  |-->|     NTP Server     |-->|     Router R1      |-->| Switch/Router noi bo |
|    (Stratum 0)     |   |    (Stratum 1)     |   |    (Stratum 2)     |   |     (Stratum 3)      |
+--------------------+   +--------------------+   +--------------------+   +----------------------+
```

**Stratum** thể hiện "khoảng cách phân cấp" của thiết bị so với nguồn thời gian chuẩn (Stratum 0 — đồng hồ nguyên tử/GPS, không phải thiết bị mạng): thiết bị đồng bộ trực tiếp từ Stratum 0 trở thành Stratum 1, thiết bị đồng bộ từ Stratum 1 trở thành Stratum 2, và cứ thế tăng dần tới tối đa Stratum 15 (Stratum 16 nghĩa là "chưa đồng bộ, không có nguồn thời gian hợp lệ").

> [!question] Nếu router cấu hình nhiều `ntp server` cùng lúc, router chọn theo tiêu chí nào?
> Router ưu tiên NTP server có **Stratum thấp nhất** (gần nguồn chuẩn nhất) trong số các server khai báo, đồng thời cân nhắc thêm độ ổn định và độ trễ round-trip. Khai báo nhiều NTP server cùng lúc là thực hành chuẩn để có dự phòng — nếu 1 server ngừng phản hồi, router tự động chuyển sang dùng server còn lại mà không cần can thiệp tay.

**Bật xác thực NTP** (khuyến nghị trong môi trường doanh nghiệp để tránh bị giả mạo nguồn thời gian):

```
Router(config)# ntp authenticate
Router(config)# ntp authentication-key 1 md5 Ntp@2026
Router(config)# ntp trusted-key 1
Router(config)# ntp server 10.10.10.100 key 1
```

## 3. Kiểm tra đồng bộ thời gian

```
Router# show clock detail
Router# show ntp status
Router# show ntp associations
```

`show ntp status` cho biết router đã đồng bộ thành công hay chưa (`Clock is synchronized`) và Stratum hiện tại. `show ntp associations` liệt kê toàn bộ NTP server đã khai báo cùng trạng thái từng server (server nào đang được dùng làm nguồn chính, ký hiệu `*`).

## 4. Log trên Router/Switch: quản lý, xem, phân tích

Mặc định, log được lưu tạm trong bộ nhớ RAM (buffer) và hiển thị ngay trên console — nhưng buffer có giới hạn dung lượng, log cũ sẽ bị ghi đè khi đầy.

```
Router# show logging
Router(config)# logging buffered 16384
```

`logging buffered 16384` tăng kích thước buffer log lên 16KB (mặc định thường nhỏ hơn nhiều) — giúp giữ được nhiều sự kiện hơn trước khi bị ghi đè, nhưng vẫn chỉ là **giải pháp tạm**, vì log biến mất hoàn toàn nếu thiết bị khởi động lại (buffer nằm trong RAM). Đây chính là lý do cần Syslog Server (mục 5) để lưu log **bền vững, ngoài thiết bị**.

**Xem log khi truy cập từ xa (Telnet/SSH):**

```
Router# terminal monitor
```

> [!question] Vì sao gõ `debug` hoặc chờ log xuất hiện qua phiên SSH mà không thấy gì, dù trên console vẫn hiện bình thường?
> Vì mặc định, Cisco IOS chỉ tự động gửi log **ra console vật lý**, không tự gửi tới các phiên truy cập từ xa (Telnet/SSH/VTY) dù đang đăng nhập đúng quyền — đây là hành vi thiết kế, không phải lỗi. Lệnh `terminal monitor` bật tính năng gửi log tới phiên VTY hiện tại; thiếu lệnh này là nguyên nhân rất phổ biến khiến người mới tưởng log/debug "không hoạt động" khi làm việc qua SSH.

**Phân tích/lọc log theo từ khóa** — hữu ích khi log quá nhiều, cần tìm nhanh sự kiện liên quan 1 interface hoặc giao thức cụ thể:

```
Router# show logging | include GigabitEthernet0/1
Router# show logging | include %LINK
```

Toán tử `| include <từ khóa>` lọc chỉ hiển thị các dòng log chứa đúng từ khóa đó — ví dụ lọc riêng log liên quan tới 1 interface, hoặc riêng nhóm log `%LINK` (thay đổi trạng thái interface, mức 3 `%LINK-3-UPDOWN`; còn trạng thái line protocol là `%LINEPROTO-5-UPDOWN`, mức 5), giúp phân tích nhanh thay vì đọc thủ công toàn bộ buffer.

## 5. Cài đặt Syslog Server, giám sát log tập trung

```
Syslog client gui log ve Syslog Server:
+----------+   +----------+   +----------+
|  Router  |   |  Switch  |   | Firewall |
+----------+   +----------+   +----------+
     |              |              |
          \--------------------/
          +--------------------+
          |   SYSLOG SERVER    |
          |  (log tap trung)   |
          +--------------------+
```

```
Router(config)# logging host 10.10.10.200
Router(config)# logging source-interface loopback0
Router(config)# logging trap informational
```

- `logging host <IP>`: chỉ định địa chỉ Syslog Server sẽ nhận log.
- `logging source-interface loopback0`: chỉ định địa chỉ nguồn gửi log là Loopback (ổn định, không phụ thuộc trạng thái vật lý — cùng lý do dùng Loopback làm Router ID đã học ở [[Bai-14_OSPF|Bài 14]] mục 2), giúp Syslog Server dễ nhận diện đúng thiết bị nào gửi log dù thiết bị đó có nhiều interface.
- `logging trap informational`: chỉ gửi log từ mức Informational (severity 6) trở lên nghiêm trọng hơn ra Syslog Server (chi tiết mức độ ở mục 6).

> [!question] Vì sao nên dùng địa chỉ Loopback làm `logging source-interface` thay vì để mặc định (IP của interface gửi log ra ngoài)?
> Vì mặc định, địa chỉ nguồn trong gói Syslog sẽ là IP của **interface vật lý** mà gói tin đi ra — nếu router có nhiều đường ra Syslog Server (dự phòng định tuyến), địa chỉ nguồn có thể **thay đổi** tùy theo đường nào đang được dùng tại thời điểm đó, khiến Syslog Server khó nhận diện chính xác log đến từ thiết bị nào một cách nhất quán. Loopback không bao giờ down (trừ khi bị xóa cấu hình), nên địa chỉ nguồn luôn cố định bất kể đường vật lý nào đang hoạt động.

## 6. Phân loại mức độ log (Severity Level)

Cisco IOS dùng 8 mức độ nghiêm trọng (0 = nghiêm trọng nhất, 7 = ít nghiêm trọng nhất), kế thừa chuẩn Syslog:

| Mức | Tên | Ý nghĩa |
|---|---|---|
| 0 | Emergency | Hệ thống không thể sử dụng được |
| 1 | Alert | Cần khắc phục ngay lập tức |
| 2 | Critical | Lỗi nghiêm trọng ở mức phần cứng/hệ thống |
| 3 | **Error** | Lỗi xảy ra nhưng thiết bị vẫn hoạt động |
| 4 | **Warning** | Cảnh báo tình huống tiềm ẩn vấn đề |
| 5 | **Notice** | Sự kiện đáng chú ý nhưng không phải lỗi (ví dụ `%LINEPROTO-5-UPDOWN` line protocol up/down; lưu ý `%LINK-3-UPDOWN` là mức 3 Error) |
| 6 | **Informational** | Thông tin chung, thường dùng khi troubleshoot |
| 7 | Debugging | Chi tiết gỡ lỗi sâu nhất, chỉ dùng tạm thời khi debug |

> [!question] Nếu cấu hình `logging trap warning` (mức 4), Syslog Server sẽ nhận được những log ở mức nào?
> Nhận được **toàn bộ log từ mức 4 trở lên mức độ nghiêm trọng hơn**, tức là các mức 0, 1, 2, 3, và 4 — **không** nhận các mức 5, 6, 7 (ít nghiêm trọng hơn). Cơ chế này áp dụng cho mọi cấu hình `logging trap <mức>`: khai báo 1 mức nghĩa là "gửi mức này và mọi mức nghiêm trọng hơn nó", không phải chỉ đúng 1 mức đó.

## 7. Cấu hình Notification cảnh báo

Ngoài lưu log thụ động, có thể cấu hình chủ động cảnh báo khi có sự kiện đạt mức độ nhất định — cách phổ biến nhất là kết hợp Syslog Server với hệ thống giám sát (ví dụ Zabbix ở Module Monitoring & Backup) để tự động gửi email/thông báo khi log khớp điều kiện đã định nghĩa, thay vì quản trị viên phải chủ động ngồi xem log liên tục.

```
Router(config)# logging trap warning
Router(config)# logging host 10.10.10.200
Router(config)# snmp-server enable traps
Router(config)# snmp-server host 10.10.10.200 version 2c public
```

`snmp-server enable traps` bật cơ chế gửi SNMP trap — 1 dạng cảnh báo tức thời riêng biệt với Syslog, thường được hệ thống giám sát dùng để kích hoạt cảnh báo (alert) theo thời gian thực khi có sự kiện quan trọng, thay vì phải liên tục "hỏi thăm" (poll) trạng thái thiết bị.

## 8. Case study — NTP Amplification DDoS (2014): rủi ro bảo mật của NTP cấu hình sai

> [!example] Case study — Lỗ hổng "monlist" trong NTP bị lợi dụng cho tấn công DDoS diện rộng
> Đầu năm 2014, CISA (khi đó là US-CERT) ghi nhận hàng loạt cuộc tấn công DDoS diện rộng khai thác lệnh `monlist` (Mode 7) trên các NTP server cấu hình mặc định không tắt tính năng này (CVE-2013-5211). Lệnh `monlist` trả về danh sách 600 truy vấn gần nhất mà NTP server đã xử lý — kích thước phản hồi có thể lớn gấp tới **5.500 lần** kích thước gói yêu cầu ban đầu (theo tài liệu của Cisco). Kẻ tấn công giả mạo địa chỉ IP nguồn (source spoofing) thành địa chỉ của nạn nhân, gửi hàng loạt yêu cầu `monlist` nhỏ tới các NTP server công khai bị lỗi — các server này vô tình trở thành công cụ khuếch đại traffic, dội một lượng dữ liệu khổng lồ vào đúng địa chỉ nạn nhân. Một NTP server bị lợi dụng ghi nhận có thể tạo ra lượng traffic rất lớn đổ về nạn nhân chỉ từ 1 thiết bị duy nhất (con số ~500 Mbps từng nêu chưa có nguồn xác nhận).
>
> **Bài học thực chiến**: dù đây là lỗ hổng trên NTP server công khai (khác với NTP client trong nội bộ doanh nghiệp học ở bài này), nguyên lý rút ra vẫn áp dụng trực tiếp: bất kỳ dịch vụ nào trả lời truy vấn UDP không cần xác thực đều tiềm ẩn nguy cơ bị lợi dụng làm bàn đạp tấn công nếu để lộ ra Internet không kiểm soát. Đây là lý do NTP server nội bộ doanh nghiệp không nên public trực tiếp ra Internet, nên bật xác thực (mục 2), và nên giới hạn truy cập bằng ACL ([[Bai-17_Advanced-ACL|Bài 17]]) chỉ cho phép các thiết bị nội bộ truy vấn. Nguồn (CVE-2013-5211, `monlist`, 600 địa chỉ): [CISA — NTP Amplification Attacks Using CVE-2013-5211](https://www.cisa.gov/news-events/alerts/2014/01/13/ntp-amplification-attacks-using-cve-2013-5211); hệ số khuếch đại 5.500 lần: [Cisco — NTP Amplification DDoS](https://sec.cloudapps.cisco.com/security/center/resources/network_time_protocol_amplification_ddos.html).

## 9. Verify & Troubleshoot

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| `show ntp status` báo "Clock is unsynchronized" kéo dài | NTP server không phản hồi (sai IP, chặn bởi ACL/firewall), hoặc chênh lệch thời gian ban đầu quá lớn khiến NTP từ chối đồng bộ | Kiểm tra kết nối tới NTP server (`ping`), đối chiếu ACL có đang chặn UDP port 123 hay không |
| Log trên Syslog Server có timestamp sai lệch giữa các thiết bị | Thiết bị chưa đồng bộ NTP đúng cách, hoặc múi giờ (`clock timezone`) khai báo sai/không nhất quán | Đối chiếu `show clock detail` và `show ntp status` trên từng thiết bị, đảm bảo cùng nguồn NTP và cùng cấu hình múi giờ |
| Syslog Server không nhận được log dù thiết bị vẫn hoạt động bình thường | Sai `logging host`, hoặc `logging trap` đặt ở mức quá nghiêm trọng (ví dụ chỉ level 0-1) khiến các sự kiện thường gặp không được gửi | Kiểm tra `logging host` đúng IP; hạ mức `logging trap` xuống mức phù hợp hơn (ví dụ informational) để xác nhận luồng log hoạt động |
| Buffer log trên thiết bị luôn trống dù vừa xảy ra sự kiện | Buffer quá nhỏ, log cũ bị ghi đè quá nhanh trước khi kịp xem (mục 4) | Tăng `logging buffered <kích thước>` hợp lý, đồng thời đảm bảo đã cấu hình Syslog Server để không phụ thuộc hoàn toàn vào buffer RAM |

## 10. Lab thực hành

**Môi trường**: Cisco Packet Tracer (hoặc GNS3 nếu cần mô phỏng Syslog Server thật), 1 router, 1 Syslog Server giả lập.

1. Cấu hình múi giờ và đồng bộ NTP theo mục 2 (nếu Packet Tracer không mô phỏng đầy đủ NTP, dùng `clock set` thủ công để minh họa khái niệm).
2. Cấu hình `logging host` trỏ về Syslog Server, đặt `logging trap informational` theo mục 5-6.
3. Tạo 1 sự kiện log thật (ví dụ `shutdown`/`no shutdown` 1 interface), xác nhận log xuất hiện trên Syslog Server với đúng timestamp.
4. Đổi `logging trap` xuống mức `warning`, tạo lại sự kiện tương tự, so sánh xem log có còn xuất hiện trên Syslog Server hay không — liên hệ cơ chế "gửi mức này và nghiêm trọng hơn" ở mục 6. Lưu ý: `shutdown`/`no shutdown` sinh `%LINK-3-UPDOWN` (mức 3) và `%LINEPROTO-5-UPDOWN` (mức 5); khi đặt `warning` (mức 4) thì `%LINK-3-UPDOWN` vẫn được gửi, còn `%LINEPROTO-5-UPDOWN` thì không.

**Kết quả kỳ vọng**: hiểu vì sao NTP và Syslog luôn đi cùng nhau trong vận hành thực tế, thành thạo cấu hình cả 2, và nắm rõ cơ chế lọc theo severity level.

## 11. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| `logging trap warning` chỉ gửi đúng các log ở mức Warning | Nhầm "chọn 1 mức" với "chọn ngưỡng tối thiểu nghiêm trọng" | Gửi mọi log từ mức đã chọn trở lên mức độ nghiêm trọng hơn (mục 6) |
| Log lưu trong buffer RAM là đủ, không cần Syslog Server | Bỏ qua rủi ro mất log khi thiết bị khởi động lại | Buffer chỉ là lưu tạm, cần Syslog Server để lưu log bền vững ngoài thiết bị (mục 4) |
| NTP chỉ đơn thuần là tiện ích "xem đúng giờ", không liên quan bảo mật | Bỏ qua rủi ro như case study mục 8 và tầm quan trọng với điều tra sự cố | NTP cấu hình sai/không xác thực có thể bị lợi dụng tấn công (mục 8), và đồng bộ sai ảnh hưởng trực tiếp tới độ tin cậy của log khi điều tra sự cố (mục 1) |
| Múi giờ chỉ là "hiển thị cho dễ đọc", không ảnh hưởng gì tới NTP | Nhầm lẫn giữa cách hiển thị và cách đồng bộ thực tế | NTP luôn truyền tải theo UTC; `clock timezone` chỉ ảnh hưởng cách hiển thị cục bộ, nhưng khai báo sai/không nhất quán giữa các thiết bị vẫn gây khó khăn khi đối chiếu log giữa chúng |

## 12. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao NTP luôn được xem là bước cấu hình bắt buộc trước khi triển khai Syslog tập trung?**
→ Vì giá trị cốt lõi của Syslog tập trung là đối chiếu trình tự sự kiện giữa nhiều thiết bị — nếu đồng hồ các thiết bị không đồng bộ, timestamp trong log sai lệch khiến việc dựng lại đúng trình tự sự kiện khi điều tra sự cố trở nên không đáng tin cậy.

**2. Stratum trong NTP là gì?**
→ Stratum thể hiện khoảng cách phân cấp của 1 thiết bị so với nguồn thời gian chuẩn (Stratum 0). Thiết bị đồng bộ trực tiếp từ Stratum 0 là Stratum 1, thiết bị đồng bộ từ Stratum 1 là Stratum 2, và tăng dần — Stratum càng thấp càng được ưu tiên làm nguồn tin cậy hơn.

**3. Có 8 mức độ log trong Syslog, hãy liệt kê ý nghĩa 3 mức Error, Warning, Notice.**
→ Error (mức 3): lỗi xảy ra nhưng thiết bị vẫn hoạt động. Warning (mức 4): cảnh báo tình huống tiềm ẩn vấn đề, chưa hẳn là lỗi. Notice (mức 5): sự kiện đáng chú ý nhưng không phải lỗi, ví dụ line protocol của interface up/down (`%LINEPROTO-5-UPDOWN`); còn `%LINK-3-UPDOWN` là mức 3 (Error).

**4. Vì sao nên dùng địa chỉ Loopback làm `logging source-interface` thay vì để mặc định?**
→ Vì địa chỉ mặc định phụ thuộc vào interface vật lý đang gửi gói tin ra, có thể thay đổi khi có nhiều đường dự phòng, khiến Syslog Server khó nhận diện nhất quán thiết bị gửi log. Loopback luôn ổn định, không phụ thuộc trạng thái vật lý.

**5. Lỗ hổng NTP amplification (monlist) khai thác cơ chế nào để tạo ra tấn công DDoS?**
→ Khai thác việc lệnh `monlist` trả về phản hồi lớn hơn nhiều lần (tới 5.500 lần theo tài liệu Cisco) so với gói yêu cầu, kết hợp với giả mạo địa chỉ IP nguồn thành địa chỉ nạn nhân — khiến NTP server vô tình trở thành công cụ khuếch đại traffic dội vào nạn nhân.

## 13. Tài liệu tham khảo

- [NetsTuts — NTP (Network Time Protocol) Explained – Stratum Levels, Cisco IOS Configuration](https://www.netstuts.com/ntp-sync)
- [NetworkLessons — Cisco IOS Syslog Messages](https://networklessons.com/system-management/cisco-ios-syslog-messages)
- [FlackBox — Cisco Syslog Tutorial](https://www.flackbox.com/cisco-syslog-tutorial)
- [CISA — NTP Amplification Attacks Using CVE-2013-5211](https://www.cisa.gov/news-events/alerts/2014/01/13/ntp-amplification-attacks-using-cve-2013-5211)

---
**Liên kết**: [[Bai-17_Advanced-ACL|← Bài 17]] · [[Bai-19_Dich-vu-DHCP|Bài 19 - Dịch vụ DHCP →]]
