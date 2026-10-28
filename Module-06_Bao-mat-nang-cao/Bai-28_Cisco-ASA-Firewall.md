---
title: Bài 28 - Bảo vệ hệ thống với Firewall ASA
type: Configuration
difficulty: Advanced
lab_required: true
exam: "Sysadmin Mid-level — Firewall chuyên dụng là lớp bảo vệ cuối cùng ở biên mạng, và hiểu đúng Security Level của ASA là kiến thức thực chiến bắt buộc khi vận hành thiết bị Cisco"
tags: [networking, security, firewall, asa, ngfw, ips, ids, proxy, sysadmin]
aliases: [ASA, Cisco ASA, Firewall, NGFW, Security Level, Proxy Firewall]
related: ["[[Bai-16_Basic-ACL]]", "[[Bai-20_NAT]]", "[[Bai-22_Remote-Access-VPN]]", "[[Bai-26_Bao-mat-Lop-2]]", "[[Bai-27_AAA]]"]
last_review: 2026-07-21
---

# Bài 28: Bảo vệ hệ thống với Firewall ASA

> [!important] Tại sao cần file này
> [[Bai-16_Basic-ACL|Bài 16]]/[[Bai-17_Advanced-ACL|Bài 17]] đã học ACL trên router — công cụ lọc traffic **stateless** (không theo dõi trạng thái kết nối). [[Bai-20_NAT|Bài 20]] và [[Bai-22_Remote-Access-VPN|Bài 22]] đều từng nhắc "chi tiết đầy đủ về ASA sẽ học ở Bài 28" — đây chính là bài đó. **Cisco ASA (Adaptive Security Appliance)** là thiết bị firewall chuyên dụng, bổ sung khả năng lọc **stateful** (theo dõi trạng thái từng kết nối) và mô hình bảo mật theo **Security Level** — khác hẳn cách ACL trên router hoạt động.

> **Lưu ý cập nhật:** dòng ASA 5500-X cũ đã End-of-Sale; Cisco hiện gọi sản phẩm này là Secure Firewall ASA và khuyến nghị Secure Firewall Threat Defense (FTD) cho các triển khai mới. Cú pháp ASA trong bài vẫn dùng để học nguyên lý Security Level, NAT và Access Rules.

## 1. Giới thiệu Firewall và các mô hình triển khai

**Firewall** là thiết bị/phần mềm đứng giữa 2 vùng mạng có mức độ tin cậy khác nhau, kiểm soát traffic được phép đi qua theo 1 tập luật đã định nghĩa. Xét theo **hình thức triển khai**, firewall có thể là: **Hardware Firewall** (thiết bị vật lý chuyên dụng, đặt tại biên mạng — như ASA), **Software Firewall** (phần mềm cài trên máy chủ/endpoint, ví dụ Windows Defender Firewall), hoặc **Cloud Firewall** (dịch vụ firewall vận hành trên nền tảng đám mây, bảo vệ tài nguyên cloud).

Xét theo **kiến trúc lọc traffic**, firewall được phân loại theo mức độ "thông minh" tăng dần:

| Kiến trúc | Cơ chế | Tầng OSI xử lý |
|---|---|---|
| **Packet-Filtering Firewall** | Lọc từng gói tin độc lập theo IP/Port/Protocol, không theo dõi trạng thái kết nối (tương tự ACL Bài 16/17) | Lớp 3-4 |
| **Circuit-Level Gateway** | Kiểm tra tính hợp lệ của quá trình bắt tay (handshake) TCP/UDP, không kiểm tra nội dung gói tin | Lớp 5 (Session) |
| **Stateful Inspection Firewall** | Theo dõi đầy đủ trạng thái từng kết nối qua Connection Table (như ASA — mục 2) | Lớp 3-4, có ngữ cảnh kết nối |
| **Proxy Firewall (Application Gateway)** | Đóng vai trò trung gian thực sự — nhận request từ client, tự tạo 1 kết nối mới tới server thay mặt client, kiểm tra được cả nội dung ứng dụng | Lớp 7 (Application) |
| **NGFW** | Stateful + Deep Packet Inspection + nhận diện ứng dụng + tích hợp sẵn IPS | Lớp 3-7 |

> [!question] Vì sao Proxy Firewall được xem là an toàn hơn Packet-Filtering hay Stateful Firewall thông thường, dù tốc độ xử lý chậm hơn đáng kể?
> Vì Proxy Firewall không đơn thuần "cho gói tin đi qua" — nó **chấm dứt kết nối gốc từ client tại chính firewall**, kiểm tra toàn bộ nội dung ở tầng ứng dụng, rồi mới **tự khởi tạo 1 kết nối hoàn toàn mới** tới server thật thay mặt client. Điều này có nghĩa là client và server **không bao giờ giao tiếp trực tiếp** với nhau — ngay cả khi gói tin gốc chứa mã độc khai thác lỗ hổng tầng ứng dụng, server thật chỉ nhận được kết nối đã qua xử lý lại từ Proxy. Đánh đổi là Proxy Firewall phải xử lý ở tầng ứng dụng (tầng 7) tốn nhiều tài nguyên hơn nhiều so với chỉ xem header ở tầng 3-4 như Packet-Filtering/Stateful Firewall, nên tốc độ xử lý chậm hơn đáng kể.

## 2. Vì sao cần Firewall chuyên dụng — Stateless, Stateful và NGFW

ACL trên router (Bài 16/17) là **stateless** (packet-filtering, mục 1) — mỗi gói tin được xét độc lập theo đúng luật đã khai báo, router không nhớ gì về các gói tin trước đó thuộc cùng 1 kết nối (từ khóa `established` ở Bài 17 chỉ là giải pháp tình thế, dựa trên việc kiểm tra cờ TCP ACK/RST chứ không thực sự theo dõi trạng thái đầy đủ). Firewall chuyên dụng như ASA hoạt động **stateful** — theo dõi đầy đủ trạng thái từng kết nối (địa chỉ, port, số thứ tự gói tin, giai đoạn bắt tay TCP) trong 1 bảng gọi là **Connection Table**, tự động cho phép traffic phản hồi hợp lệ của 1 kết nối đã được phép đi ra, mà không cần khai báo thêm luật ngược chiều thủ công như ACL. Lưu ý: cơ chế tự cho phép phản hồi này mặc định chỉ áp dụng cho TCP/UDP; ICMP echo reply bị chặn trừ khi bật `inspect icmp` trong policy-map (`global_policy` mặc định không có).

| Loại | Cơ chế | Hạn chế |
|---|---|---|
| **Stateless (ACL router)** | Xét từng gói tin độc lập theo luật cố định | Không hiểu ngữ cảnh kết nối, dễ bị khai thác bằng gói tin giả mạo cờ TCP |
| **Stateful (ASA)** | Theo dõi trạng thái đầy đủ từng kết nối qua Connection Table | Chủ yếu xét ở Lớp 3/4 (IP, port); có kiểm tra ứng dụng ở mức giới hạn qua Modular Policy Framework (`inspect dns/ftp/sip/http`...) nhưng chưa nhận diện ứng dụng sâu như NGFW |
| **NGFW (Next-Generation Firewall)** | Stateful + Deep Packet Inspection, nhận diện ứng dụng thực tế, tích hợp sẵn IPS | Yêu cầu tài nguyên xử lý cao hơn, chi phí cao hơn |

**"Traditional Firewall"** (tên gọi chung cho Packet-Filtering/Stateful thế hệ trước) chỉ nhìn vào IP/Port/Protocol để ra quyết định — hoàn toàn không biết bên trong gói tin đang chở ứng dụng gì. **NGFW** giải quyết đúng điểm mù này bằng khả năng nhận diện **chính xác ứng dụng thực tế** (ví dụ phân biệt được Zoom, Dropbox, hay 1 ứng dụng độc hại) bất kể chúng dùng port/giao thức nào để ngụy trang, đồng thời tích hợp sẵn IPS (mục 8) ngay trong 1 thiết bị duy nhất thay vì phải triển khai rời.

## 3. Kiến trúc ASA — Interface, Security Level, nameif

Khác với router (nơi mọi interface bình đẳng, kiểm soát hoàn toàn bằng ACL), ASA gán cho mỗi interface 1 **tên logic** (`nameif`) và 1 **Security Level** (0-100, càng cao càng tin cậy):

```
+--------------+  +--------------+  +--------------+
|    INSIDE    |  |     ASA      |  |   OUTSIDE    |
|  Level 100   |  |   (nameif)   |  |   Level 0    |
+--------------+  +--------------+  +--------------+
```

| Interface | Tên (nameif) thường dùng | Security Level thường dùng | Ý nghĩa |
|---|---|---|---|
| Hướng mạng nội bộ | `inside` | 100 (cao nhất) | Vùng tin cậy nhất |
| Hướng Internet | `outside` | 0 (thấp nhất) | Vùng không tin cậy |
| Vùng trung gian (server công khai) | `dmz` | 50 (hoặc giá trị trung gian tùy chọn) | Tin cậy hơn Internet nhưng thấp hơn mạng nội bộ |

```
ciscoasa(config)# interface gigabitEthernet 0/0
ciscoasa(config-if)# nameif inside
ciscoasa(config-if)# security-level 100
ciscoasa(config-if)# ip address 10.10.10.1 255.255.255.0

ciscoasa(config)# interface gigabitEthernet 0/1
ciscoasa(config-if)# nameif outside
ciscoasa(config-if)# security-level 0
ciscoasa(config-if)# ip address 203.0.113.10 255.255.255.0
```

**Quy tắc mặc định của Security Level** (trước khi có bất kỳ ACL nào): traffic đi từ Security Level **cao xuống thấp** (inside → outside) **mặc định được phép**; traffic đi từ Security Level **thấp lên cao** (outside → inside) **mặc định bị chặn**.

> **Lưu ý cập nhật:** interface vật lý của ASA mặc định ở trạng thái shutdown, nên mỗi interface trong ví dụ trên cần thêm `no shutdown`. Ngoài ra, nếu hai interface có cùng Security Level thì traffic giữa chúng mặc định bị chặn; cần bật `same-security-traffic permit inter-interface` để cho phép.

> [!question] Vì sao ASA cần thêm khái niệm Security Level, trong khi router chỉ cần ACL là đủ kiểm soát hướng traffic?
> Vì đây là triết lý thiết kế "an toàn theo mặc định" (secure by default) đặc trưng của thiết bị firewall chuyên dụng — ngay cả khi quản trị viên **quên** cấu hình ACL, ASA vẫn tự động chặn traffic từ vùng không tin cậy (outside) đi vào vùng tin cậy (inside) nhờ cơ chế Security Level. Router thông thường không có khái niệm "vùng tin cậy" mặc định — nếu quên cấu hình ACL, router sẽ cho phép **mọi traffic** đi qua tự do theo đúng routing table, không có lớp bảo vệ ngầm định nào. Đây là lý do ASA phù hợp hơn để đặt ở vị trí biên mạng (edge), nơi hậu quả của việc quên cấu hình đúng là nghiêm trọng nhất.

## 4. Access Rules trên ASA

Dù Security Level đã cho phép/chặn theo mặc định, ACL (gọi là **Access Rules** trên ASA, cấu hình qua ASDM hoặc CLI) vẫn cần thiết để: cho phép traffic từ vùng thấp lên vùng cao (ví dụ Internet truy cập server web ở `dmz`, hoặc VPN Client-to-Site từ Bài 22 truy cập tài nguyên `inside`), hoặc thắt chặt thêm traffic từ vùng cao xuống thấp (mặc định được phép, nhưng có thể muốn giới hạn cụ thể hơn).

```
ciscoasa(config)# access-list OUTSIDE-IN extended permit tcp any host 10.10.50.10 eq 80
ciscoasa(config)# access-group OUTSIDE-IN in interface outside
```

Cấu hình trên cho phép traffic HTTP (port 80) từ **bất kỳ đâu** (`any`) tới đúng địa chỉ thật `10.10.50.10` của server DMZ (từ ASA 8.3 trở lên, ASA un-NAT gói tin trước khi kiểm tra ACL nên ACL phải dùng IP thật của server, không dùng IP Public đã ánh xạ NAT) — dù Security Level mặc định chặn hướng outside→inside/dmz, `access-group` áp ACL này vào interface `outside` sẽ **ghi đè** đúng phần traffic đã khai báo permit.

> [!question] Khi có xung đột giữa Security Level và Access Rules, cái nào được ưu tiên?
> Access Rules (ACL áp trực tiếp vào interface qua `access-group`) luôn được ưu tiên hơn quy tắc mặc định của Security Level. Điều này hợp lý vì Security Level chỉ là hành vi **mặc định** khi không có chỉ định cụ thể nào khác — còn ACL là chỉ định **tường minh** của quản trị viên về đúng loại traffic nào được phép, nên luôn có độ ưu tiên cao hơn hành vi ngầm định.

## 5. Cấu hình Route trên ASA

ASA cũng cần định tuyến để biết traffic phải đi ra đúng interface/next-hop nào — đơn giản nhất là Static Route, cú pháp tương tự router (đã học ở [[Bai-13_Static-Route|Bài 13]]) nhưng có thêm tham số `nameif`:

```
ciscoasa(config)# route outside 0.0.0.0 0.0.0.0 203.0.113.1
ciscoasa(config)# route inside 10.20.0.0 255.255.0.0 10.10.10.254
```

Cú pháp: `route <nameif> <destination_network> <subnet_mask> <next-hop>`. Dòng đầu khai báo **default route** ra `outside` qua gateway ISP `203.0.113.1` — mọi traffic không khớp route cụ thể nào khác sẽ đi theo đường này. Dòng sau khai báo route tới 1 mạng nội bộ khác (`10.20.0.0/16`) qua next-hop `10.10.10.254` — ví dụ khi có nhiều subnet nội bộ nằm sau 1 router khác ở phía `inside`, ASA cần biết đường tới đó thay vì chỉ coi toàn bộ `inside` là 1 mạng phẳng.

> [!question] ASA có hỗ trợ định tuyến động (OSPF/EIGRP đã học ở Module 3) không, hay chỉ dùng được Static Route?
> ASA có hỗ trợ định tuyến động (OSPF, EIGRP, BGP tùy phiên bản) tương tự router, nhưng trong thực tế triển khai, phần lớn ASA đặt ở vị trí **biên mạng** (edge) chỉ cần đúng 1-2 route cố định (default route ra ISP, và route ngược lại vào các mạng nội bộ đã biết trước) — Static Route đơn giản, dễ kiểm soát và **giảm bề mặt tấn công** (ASA không cần "nói chuyện" định tuyến với router khác, tránh rủi ro bị khai thác qua chính giao thức định tuyến). Định tuyến động trên ASA chỉ thực sự cần thiết trong các thiết kế phức tạp hơn, nơi topology phía sau ASA thay đổi thường xuyên.

## 6. NAT trên ASA

ASA cũng thực hiện NAT (đã học khái niệm cơ bản ở [[Bai-20_NAT|Bài 20]]), thường dùng để publish server trong DMZ ra Internet (tương tự Port Forwarding), hoặc PAT cho toàn bộ mạng inside ra Internet:

```
ciscoasa(config)# object network INSIDE-NET
ciscoasa(config-network-object)# subnet 10.10.10.0 255.255.255.0
ciscoasa(config-network-object)# nat (inside,outside) dynamic interface

ciscoasa(config)# object network DMZ-WEBSERVER
ciscoasa(config-network-object)# host 10.10.50.10
ciscoasa(config-network-object)# nat (dmz,outside) static 203.0.113.10
```

Câu lệnh đầu (`nat (inside,outside) dynamic interface`) tương đương PAT/NAT Overload đã học ở Bài 20 — toàn bộ mạng `10.10.10.0/24` dùng chung IP của interface `outside` khi ra Internet. Câu lệnh sau ánh xạ tĩnh 1-1 giữa server DMZ nội bộ (`10.10.50.10`) và IP Public (`203.0.113.10`) — tương đương Static NAT, là bước nền tảng cho việc publish dịch vụ ra ngoài ở mục 7.

## 7. Publish dịch vụ trên Firewall ASA

"Publish dịch vụ" là việc kết hợp **NAT tĩnh** (mục 6) và **Access Rules** (mục 4) để 1 server nội bộ (thường đặt ở DMZ) có thể được truy cập **an toàn** từ Internet, mà vẫn giữ được kiểm soát chặt chẽ về đúng dịch vụ/port nào được mở ra ngoài — đây là quy trình 2 bước bắt buộc phải làm **cùng nhau**, thiếu 1 trong 2 đều không hoạt động:

```
! Buoc 1 - NAT tinh: anh xa IP Public toi server DMZ that
ciscoasa(config)# object network DMZ-WEBSERVER
ciscoasa(config-network-object)# host 10.10.50.10
ciscoasa(config-network-object)# nat (dmz,outside) static 203.0.113.10 service tcp 80 80

! Buoc 2 - Access Rule: ASA 8.3+ un-NAT truoc khi kiem tra ACL nen ACL dung IP THAT cua server DMZ (10.10.50.10), cho phep dung port 80 di tu outside vao dmz
ciscoasa(config)# access-list OUTSIDE-IN extended permit tcp any host 10.10.50.10 eq 80
ciscoasa(config)# access-group OUTSIDE-IN in interface outside
```

Tham số `service tcp 80 80` trong lệnh NAT giới hạn ánh xạ tĩnh chỉ áp dụng cho đúng port 80 (thay vì ánh xạ toàn bộ mọi port của server) — đây là dạng **Static NAT theo Port**, tương đương Port Forwarding đã học ở [[Bai-20_NAT|Bài 20]] mục 4, nhưng trên ASA cần thêm bước Access Rule tường minh (mục 4) vì Security Level mặc định chặn hướng outside→dmz.

> [!question] Vì sao chỉ cấu hình NAT tĩnh (Bước 1) mà thiếu Access Rule (Bước 2) thì việc publish dịch vụ vẫn thất bại?
> Vì NAT chỉ làm nhiệm vụ **dịch địa chỉ** (đổi IP Public thành IP nội bộ tương ứng khi gói tin đi qua ASA) — nó hoàn toàn không liên quan tới việc **có cho phép** gói tin đó đi qua hay không. Quyết định "cho phép hay chặn" nằm ở Security Level và Access Rules (mục 3-4): dù NAT đã ánh xạ đúng, Security Level mặc định vẫn chặn traffic từ `outside` (Level 0) vào `dmz` (Level 50) nếu không có Access Rule tường minh cho phép. Đây là lý do publish dịch vụ trên ASA luôn là quy trình **2 bước độc lập nhưng bắt buộc đi cùng nhau** — khác với NAT trên router thông thường (Bài 20) nơi chỉ cần đúng lệnh NAT là đủ, vì router không có khái niệm Security Level chặn mặc định.

## 8. So sánh IDS, IPS, Proxy và NGFW

| Công nghệ | Vị trí hoạt động | Hành vi khi phát hiện tấn công |
|---|---|---|
| **IDS** (Intrusion Detection System) | Ngoài luồng (out-of-band) — nhận bản sao traffic để phân tích | Chỉ **cảnh báo** (alert/log), không tự chặn traffic gốc |
| **IPS** (Intrusion Prevention System) | Trong luồng (inline) — traffic thực sự đi qua thiết bị | **Chủ động chặn** traffic độc hại ngay khi phát hiện |
| **Proxy** (đã nêu ở mục 1) | Trong luồng, chấm dứt và khởi tạo lại kết nối ở tầng ứng dụng | Kiểm tra nội dung tầng 7 trước khi cho phép, client/server không giao tiếp trực tiếp |
| **NGFW** | Trong luồng, tích hợp sẵn firewall + IPS + nhận diện ứng dụng | Kết hợp lọc theo Security Level/ACL truyền thống **và** kiểm tra sâu nội dung gói tin (Deep Packet Inspection) |

> [!question] Vì sao IDS chỉ cảnh báo mà không tự chặn, trong khi IPS/NGFW lại chặn được ngay?
> Vì khác biệt nằm ở **vị trí triển khai vật lý**: IDS nhận traffic qua bản sao (thường qua cổng SPAN/mirror trên switch), traffic gốc đã đi qua và tới đích **trước khi** IDS kịp phân tích xong — nên IDS chỉ có thể cảnh báo sau khi sự việc đã xảy ra. IPS/NGFW đặt **trực tiếp trên đường đi** (inline) của traffic thật — mọi gói tin đều phải đi qua thiết bị trước khi tới đích, cho phép thiết bị **chủ động giữ lại và chặn** gói tin ngay khi phát hiện dấu hiệu tấn công, trước khi nó gây hại. Đánh đổi là IPS/NGFW inline có thể trở thành điểm lỗi đơn (Single Point of Failure) hoặc điểm nghẽn hiệu năng nếu thiết bị gặp sự cố, trong khi IDS out-of-band không ảnh hưởng trực tiếp tới luồng traffic chính.

## 9. Giám sát sự kiện, hoạt động của Firewall ASA

Firewall không phải "cấu hình 1 lần rồi bỏ đó" — cần giám sát liên tục để phát hiện bất thường và phục vụ điều tra sau sự cố (tương tự tinh thần Syslog đã học ở [[Bai-18_NTP-va-Syslog|Bài 18]]).

```
ciscoasa(config)# logging enable
ciscoasa(config)# logging host inside 10.10.10.200
ciscoasa(config)# logging trap informational
ciscoasa(config)# logging buffered informational
```

`logging enable` bật tính năng ghi log tổng thể trên ASA. `logging host inside 10.10.10.200` gửi log ra 1 Syslog Server tập trung ở địa chỉ `10.10.10.200` (qua interface `inside`) — quan trọng vì log lưu cục bộ trên ASA có dung lượng giới hạn và sẽ mất nếu thiết bị gặp sự cố, trong khi Syslog Server tập trung lưu trữ lâu dài và cho phép đối chiếu log từ nhiều thiết bị cùng lúc. `logging trap informational` chỉ định mức độ chi tiết log gửi ra Syslog Server.

```
ciscoasa# show logging
ciscoasa# show conn
ciscoasa# show xlate
ciscoasa# show access-list OUTSIDE-IN
```

`show logging` xem log đang lưu tại buffer cục bộ ngay trên ASA. `show conn` liệt kê toàn bộ kết nối đang active trong Connection Table (mục 2) — công cụ quan trọng nhất để xác nhận traffic có thực sự đi qua ASA đúng như kỳ vọng hay không tại thời điểm hiện tại.

> [!question] Vì sao nên gửi log ra Syslog Server tập trung thay vì chỉ dựa vào `show logging` xem trực tiếp trên ASA?
> Vì log lưu cục bộ trên ASA (buffer) có dung lượng **giới hạn** — log cũ sẽ tự động bị ghi đè khi buffer đầy, và toàn bộ log sẽ **mất hoàn toàn** nếu thiết bị bị reload hoặc gặp sự cố đúng lúc cần điều tra nhất. Gửi log ra Syslog Server tập trung giải quyết cả 2 vấn đề: lưu trữ lâu dài không giới hạn bởi bộ nhớ của riêng ASA, và cho phép đối chiếu log từ **nhiều thiết bị khác nhau** (switch, router, ASA) trên cùng 1 dòng thời gian khi điều tra sự cố — đúng nguyên tắc tập trung hóa đã học ở Bài 18.

## 10. Defense in Depth — Tổng hợp các lớp bảo mật đã học

Case study Colonial Pipeline ở [[Bai-22_Remote-Access-VPN|Bài 22]] đã nhắc tới nguyên tắc **Defense in Depth** (phòng thủ theo chiều sâu) — không dựa vào **1 lớp bảo vệ duy nhất**, mà xếp chồng nhiều lớp độc lập để nếu 1 lớp bị vượt qua, các lớp còn lại vẫn ngăn được thiệt hại. Nhìn lại toàn bộ Module 4-6, có thể thấy rõ các lớp đó đã được xây dựng tuần tự:

| Lớp phòng thủ | Công cụ đã học | Bài |
|---|---|---|
| Lọc traffic cơ bản theo IP/Port | ACL Standard/Extended | Bài 16, 17 |
| Nhận diện đúng nguy cơ đang đối mặt | Spoofing, DoS, MITM, Buffer Overflow, Malware, Password/Human Vulnerabilities | Bài 26 |
| Bảo mật vật lý tại switchport | Port Security, DHCP Snooping, DAI, BPDU Guard/Root Guard | Bài 26 |
| Bảo mật quyền truy cập thiết bị | Mật khẩu/mã hóa, ACL VTY, AAA, RADIUS/TACACS+, 802.1X | Bài 27 |
| Mã hóa kết nối từ xa | VPN (IPsec Site-to-Site, Client-to-Site) | Bài 22 |
| Lọc stateful + NAT + giám sát ở biên mạng | Cisco ASA (bài này) | Bài 28 |

Không có lớp nào trong số này là "đủ dùng một mình" — case study Colonial Pipeline chính là minh chứng: hệ thống VPN (1 lớp) bị vượt qua chỉ vì thiếu MFA (đáng lẽ là AAA — 1 lớp khác) và tài khoản cũ không bị vô hiệu hóa (thiếu quy trình vận hành). Đây là lý do Sysadmin cần hiểu và triển khai **đồng thời nhiều lớp**, thay vì chỉ dựa vào 1 thiết bị/công nghệ duy nhất.

## 11. Verify & Troubleshoot

```
ciscoasa# show interface ip brief
ciscoasa# show nameif
ciscoasa# show route
ciscoasa# show access-list
ciscoasa# show conn
ciscoasa# show xlate
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Traffic từ inside ra outside bị chặn dù Security Level đúng thứ tự (100 → 0) | Có Access Rules tường minh đang chặn traffic đó (ACL luôn ưu tiên hơn Security Level mặc định, mục 4), hoặc thiếu route hợp lệ | `show access-list` kiểm tra có luật `deny` nào đang khớp traffic không mong muốn; `show route` xác nhận có route hợp lệ (mục 5) |
| Internet không truy cập được server DMZ dù đã cấu hình Access Rules | Thiếu NAT tương ứng (server DMZ chưa được ánh xạ ra IP Public), hoặc `access-group` chưa áp đúng interface | Đối chiếu `nat` object đã đúng chưa (mục 6-7); xác nhận `access-group` áp đúng vào interface `outside` |
| `show conn` không thấy kết nối nào dù traffic đang đi qua | Traffic có thể đang đi qua đường khác (routing sai) hoặc bị chặn trước khi tới ASA | Kiểm tra routing tổng thể (`show route`), xác nhận traffic thực sự đi qua đúng interface của ASA |
| Kết nối NAT cũ vẫn tồn tại dù đã đổi cấu hình NAT | Bảng dịch địa chỉ (`xlate`) chưa được làm mới | `show xlate` xem bản ghi hiện tại; xóa translation cũ nếu cần (tương tự `clear ip nat translation` ở Bài 20) |
| Không có log nào xuất hiện trên Syslog Server dù ASA vẫn hoạt động | Thiếu `logging enable`, hoặc sai IP/interface trong `logging host` | Kiểm tra `logging enable` đã bật chưa; đối chiếu đúng IP và interface hướng tới Syslog Server (mục 9) |

## 12. Lab thực hành

**Môi trường**: Cisco Packet Tracer hoặc GNS3, 1 ASA (hoặc router mô phỏng ASA), 3 vùng mạng: inside, dmz, outside, 1 Syslog Server mô phỏng.

1. Cấu hình 3 interface theo mục 3 với đúng `nameif`/`security-level`.
2. Cấu hình Static Route theo mục 5 (default route ra outside).
3. Xác nhận mặc định: PC ở inside ping được ra outside, nhưng ping ngược từ outside vào inside bị chặn. Lưu ý: ICMP echo reply chỉ được phép trả về khi đã bật `inspect icmp` (`policy-map global_policy` → `class inspection_default` → `inspect icmp`); nếu chưa bật thì ping từ inside cũng không nhận được reply.
4. Đặt 1 web server ở DMZ, cấu hình NAT tĩnh theo port + Access Rules theo mục 7 để publish server đó ra outside qua đúng port 80.
5. Test lại từ "Internet" giả lập (outside), xác nhận truy cập được server DMZ qua IP Public đã NAT, nhưng vẫn không truy cập trực tiếp được vào inside.
6. Cấu hình `logging host` theo mục 9 trỏ tới Syslog Server mô phỏng, xác nhận log xuất hiện khi có traffic đi qua.

**Kết quả kỳ vọng**: hiểu và thực hành đúng mô hình Security Level 3 vùng (inside/dmz/outside), thành thạo quy trình publish dịch vụ 2 bước (NAT + Access Rule), và biết cách thiết lập giám sát tập trung cho ASA.

## 13. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Có ASA rồi thì không cần ACL trên router nữa | Nhầm lẫn vai trò từng lớp trong Defense in Depth | ACL router và ASA phục vụ các vị trí/mục đích khác nhau trong mô hình nhiều lớp, không thay thế hoàn toàn cho nhau (mục 10) |
| Security Level 100/0 là quy định cứng, không đổi được | Nhầm lẫn giá trị mặc định thường dùng với giới hạn kỹ thuật | Security Level là số 0-100 do quản trị viên tự gán, `inside`/`outside` chỉ là tên gọi quy ước phổ biến chứ không bắt buộc đúng 100/0 |
| IDS và IPS là 2 tên gọi khác nhau của cùng 1 thiết bị | Bỏ qua khác biệt về vị trí triển khai (inline vs out-of-band) | IDS chỉ cảnh báo (out-of-band), IPS chủ động chặn (inline) — khác nhau về vị trí và khả năng phản ứng (mục 8) |
| NGFW chỉ là ASA/firewall thường được đổi tên marketing | Bỏ qua khác biệt kỹ thuật thực sự | NGFW tích hợp thêm Deep Packet Inspection và nhận diện ứng dụng, vượt xa khả năng lọc theo IP/Port truyền thống của firewall stateful thông thường (mục 2) |
| Chỉ cần NAT tĩnh là đủ để publish dịch vụ ra Internet | Bỏ qua vai trò độc lập của Access Rules | NAT chỉ dịch địa chỉ, không quyết định cho phép hay chặn — publish dịch vụ trên ASA luôn cần cả NAT tĩnh lẫn Access Rule tường minh đi cùng nhau (mục 7) |
| Xem log trực tiếp bằng `show logging` trên ASA là đủ, không cần Syslog Server riêng | Bỏ qua giới hạn dung lượng buffer cục bộ | Log cục bộ trên ASA có thể bị ghi đè hoặc mất khi thiết bị reload — cần Syslog Server tập trung để lưu trữ lâu dài và đối chiếu đa thiết bị (mục 9) |

## 14. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Nêu các kiến trúc lọc traffic của firewall theo mức độ "thông minh" tăng dần.**
→ Packet-Filtering (lọc từng gói độc lập theo IP/Port) → Circuit-Level Gateway (kiểm tra handshake) → Stateful Inspection (theo dõi trạng thái kết nối) → Proxy/Application Gateway (kiểm tra tầng ứng dụng, chấm dứt và khởi tạo lại kết nối) → NGFW (Stateful + DPI + nhận diện ứng dụng + IPS tích hợp).

**2. Phân biệt firewall stateless và stateful.**
→ Stateless (ACL router) xét từng gói tin độc lập theo luật cố định, không nhớ ngữ cảnh kết nối. Stateful (ASA) theo dõi đầy đủ trạng thái từng kết nối qua Connection Table, tự động cho phép traffic phản hồi TCP/UDP hợp lệ (ICMP cần thêm `inspect icmp`) mà không cần khai báo luật ngược chiều thủ công.

**3. Khi Access Rules và Security Level xung đột nhau, cái nào được ưu tiên?**
→ Access Rules (ACL áp trực tiếp vào interface) luôn được ưu tiên hơn hành vi mặc định của Security Level, vì đó là chỉ định tường minh của quản trị viên.

**4. Quy trình publish 1 dịch vụ ra Internet qua ASA gồm những bước nào, và vì sao cần cả 2?**
→ Gồm 2 bước bắt buộc đi cùng nhau: NAT tĩnh (ánh xạ IP Public tới server nội bộ, có thể giới hạn theo port) và Access Rule (cho phép tường minh traffic từ outside vào đúng port đó). NAT chỉ dịch địa chỉ chứ không quyết định cho phép hay chặn — thiếu Access Rule thì Security Level mặc định vẫn chặn traffic từ vùng thấp lên vùng cao.

**5. Vì sao nên gửi log của ASA ra Syslog Server tập trung thay vì chỉ xem `show logging` cục bộ?**
→ Vì buffer log cục bộ trên ASA có dung lượng giới hạn, log cũ bị ghi đè và mất hoàn toàn nếu thiết bị reload. Syslog Server tập trung lưu trữ lâu dài và cho phép đối chiếu log từ nhiều thiết bị khác nhau trên cùng dòng thời gian khi điều tra sự cố.

## 15. Tài liệu tham khảo

- [Cisco — Configure ASA Access Control List for Various Scenarios](https://www.cisco.com/c/en/us/support/docs/security/adaptive-security-appliance-asa-software/217679-asa-access-control-list-configuration-ex.html)
- [Grandmetric — Cisco ASA: Security Level and nameif](https://www.grandmetric.com/knowledge-base/design_and_configure/how-to-configure-security-level-and-nameif-on-cisco-asa/)
- [NetworkLessons — Cisco ASA Security Levels](https://networklessons.com/cisco/asa-firewall/cisco-asa-security-levels)
- [NetworkLessons — Cisco ASA Syslog Configuration](https://networklessons.com/cisco/asa-firewall/cisco-asa-syslog-configuration)
- [Cisco — Configuring Logging (ASA General Operations Guide)](https://www.cisco.com/c/en/us/td/docs/security/asa/asa91/configuration/general/asa_91_general_config/monitor_syslog.html)
- [Fortinet — Stateful vs Stateless Firewalls](https://www.fortinet.com/resources/cyberglossary/stateful-vs-stateless-firewall)
- [Check Point — 5 Types of Firewalls: Which One Do You Need?](https://www.checkpoint.com/cyber-hub/network-security/what-is-firewall/5-types-of-firewalls-which-one-do-you-need/)

---
**Liên kết**: [[Bai-27_AAA|← Bài 27]] · [[Bai-29_Network-Automation-va-Programmability|Bài 29 - Network Automation và Programmability →]]
