---
title: Bài 4 - Thiết bị kết nối mạng
type: Foundation
difficulty: Beginner
lab_required: true
exam: "Sysadmin Junior — bắt buộc phân biệt đúng thiết bị trước khi học cấu hình Switch/Router Cisco"
tags: [networking, fundamentals, hub, switch, router, access-point, poe, sfp, sysadmin]
aliases: [Network Devices, Thiet bi ket noi mang]
related: ["[[Bai-03_Phuong-tien-truyen-dan-mang]]", "[[Bai-01_Gioi-thieu-mang-may-tinh]]", "[[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco]]"]
last_review: 2026-07-21
---

# Bài 4: Thiết bị kết nối mạng

> [!important] Tại sao cần file này
> Mỗi loại thiết bị trung gian hoạt động ở một lớp OSI khác nhau ([[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]) và "nhìn thấy" thông tin khác nhau — Hub không đọc được địa chỉ gì cả, Switch đọc MAC, Router đọc IP. Không phân biệt rạch ròi dẫn đến sai lầm cơ bản khi thiết kế hoặc troubleshoot: đòi hỏi một thiết bị làm việc nó không có khả năng làm (ví dụ mong VLAN hoạt động trên Hub, hoặc mong Switch Layer 2 tự định tuyến giữa 2 subnet).

## 1. Bản đồ thiết bị theo lớp OSI

| Thiết bị | Lớp OSI hoạt động | Đọc được thông tin gì | Vai trò chính |
|---|---|---|---|
| Hub / Repeater | 1 — Physical | Không đọc địa chỉ, chỉ khuếch đại/lặp lại tín hiệu | Kéo dài khoảng cách tín hiệu (gần như không còn dùng) |
| Bridge | 2 — Data Link | Địa chỉ MAC (số lượng port rất hạn chế) | Tiền thân của Switch, chia nhỏ collision domain |
| Switch (Layer 2) | 2 — Data Link | Địa chỉ MAC | Chuyển frame có chọn lọc theo bảng MAC học được |
| Switch Layer 3 | 2 và 3 | MAC và IP | Vừa switch vừa định tuyến cơ bản giữa các VLAN |
| Router | 3 — Network | Địa chỉ IP | Định tuyến gói tin giữa các mạng khác nhau |
| Wireless AP | 1-2 | MAC (đóng vai trò cầu nối) | Cầu nối giữa mạng có dây và không dây |

## 2. Hub và Repeater — thiết bị Lớp 1

**Hub** nhận tín hiệu điện ở một cổng và khuếch đại, phát lại y nguyên ra **tất cả** các cổng còn lại — không đọc địa chỉ MAC, không biết frame nào cần đi đâu. Hệ quả: mọi thiết bị nối vào Hub cùng nằm chung **1 collision domain** duy nhất (giống mô hình Bus dù đi dây theo hình sao — xem lại [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]]), chỉ hoạt động half-duplex, và hiệu năng giảm mạnh khi số thiết bị tăng.

**Repeater** chức năng đơn giản hơn nữa: chỉ khuếch đại và tái tạo lại tín hiệu điện đã suy yếu để kéo dài khoảng cách truyền dẫn, không có khái niệm nhiều cổng như Hub.

Cả hai thiết bị này ngày nay gần như đã biến mất khỏi hạ tầng doanh nghiệp, được thay thế hoàn toàn bởi Switch — nhưng hiểu bản chất của Hub là nền tảng bắt buộc để hiểu **tại sao** Switch lại là một bước tiến quan trọng (mục 4).

## 3. Bridge — tiền thân của Switch

Bridge là thiết bị Lớp 2 đầu tiên biết đọc địa chỉ MAC: nó học MAC address ở mỗi cổng (thường chỉ 2-4 cổng) và chỉ chuyển tiếp frame sang đúng phân đoạn (segment) chứa địa chỉ đích, thay vì phát ra mọi hướng như Hub — nhờ đó chia nhỏ được collision domain. Bridge xử lý bằng phần mềm (software-based) nên tốc độ chậm và số cổng rất hạn chế, khiến nó không mở rộng được cho mạng lớn. Switch ra đời như một "Bridge nhiều cổng, xử lý bằng phần cứng ASIC" — nên về bản chất logic, **Switch chính là Bridge đa cổng tốc độ cao**, đây là lý do khi đọc tài liệu cũ, thuật ngữ "bridging" và "switching" thường được dùng thay thế cho nhau.

## 4. Switch — thiết bị trung tâm của LAN hiện đại

### 4.1 Cơ chế học địa chỉ MAC (MAC Learning)

Switch xây dựng và duy trì một bảng gọi là **MAC address table** (hay CAM table) ánh xạ giữa địa chỉ MAC và cổng vật lý, theo quy trình:

```
1. LEARN   — Frame vào cổng nào, switch ghi lại {MAC nguồn : cổng đó} vào bảng.
2. FLOOD   — Nếu MAC đích chưa có trong bảng, switch phát (flood)
             frame ra TẤT CẢ các cổng khác (trừ cổng nhận vào).
3. FORWARD — Nếu MAC đích đã có trong bảng, switch chỉ gửi
             frame ra ĐÚNG một cổng tương ứng — không còn flood.
4. FILTER  — Nếu MAC nguồn và đích học được trên CÙNG một cổng,
             switch không chuyển tiếp (vì 2 thiết bị đó thấy nhau trực tiếp).
```

Đây chính là lý do Switch vượt trội hoàn toàn so với Hub: mỗi cổng là một collision domain riêng, và sau khi đã "học" xong, traffic được gửi có chọn lọc thay vì phát tràn lan — vừa tăng hiệu năng vừa tăng bảo mật cơ bản (thiết bị khác không dễ dàng nghe lén traffic không dành cho mình).

### 4.2 Switch Layer 2 vs Switch Layer 3

Switch Layer 2 chỉ xử lý ở tầng Data Link (chuyển frame theo MAC). Switch Layer 3 có thêm khả năng đọc địa chỉ IP và định tuyến cơ bản — thường dùng làm lớp Distribution trong mô hình phân cấp (xem lại mục 5.3 ở [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]]) để định tuyến giữa các VLAN mà không cần một Router vật lý riêng (kỹ thuật SVI — chi tiết ở Bài 12).

> [!example] Case study — Tấn công lạm dụng Cisco Smart Install (04/2018)
> Từ 05/04/2018, hàng loạt switch Cisco trên thế giới — tập trung ở Nga và Iran — bị tấn công thông qua lỗ hổng trong tính năng **Cisco Smart Install** (giao thức hỗ trợ cấu hình tự động switch mới, không có xác thực). Kẻ tấn công lợi dụng giao thức này để ghi đè cấu hình và thay đổi banner đăng nhập của switch. Theo Cisco Talos, hơn 168.000 thiết bị công khai trên Internet (qua Shodan) bị phát hiện tồn tại lỗ hổng này; nhiều trang tin lớn của Nga (Fontanka.ru, Komsomolskaya Pravda) và dịch vụ ISP tại Iran bị gián đoạn. Bài học thực chiến: switch không chỉ là thiết bị "cắm dây là chạy" — các tính năng quản lý tiện lợi (zero-touch provisioning) nếu không tắt hoặc không kiểm soát truy cập đúng cách sẽ trở thành cửa hậu cho kẻ tấn công chiếm quyền toàn bộ thiết bị, kể cả khi mạng LAN nội bộ tưởng chừng "chỉ dùng trong công ty". Nguồn: [Tenable](https://www.tenable.com/blog/cisco-smart-install-how-to-prevent-attacks-on-switches), [Kaspersky ICS CERT](https://ics-cert.kaspersky.com/publications/news/2018/04/11/cisco-attack/).

## 5. Router — thiết bị định tuyến

Router hoạt động ở Lớp 3, đọc địa chỉ IP đích của mỗi packet, tra **bảng định tuyến (routing table)** để xác định next-hop, sau đó đóng gói lại frame Lớp 2 mới cho chặng kế tiếp (chi tiết cơ chế đã học ở mục 2.3, [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]). Khác biệt cốt lõi so với Switch: Router kết nối các **mạng khác nhau** (khác subnet/broadcast domain), trong khi Switch chỉ hoạt động trong phạm vi cùng một mạng.

> [!question] Router và Switch Layer 3 khác nhau ở đâu nếu cả hai đều định tuyến được?
> Về mặt logic định tuyến IP, chức năng tương đương. Khác biệt nằm ở phần cứng và vai trò: Router thường có ít cổng hơn nhưng đa dạng loại kết nối WAN (serial, DSL, 4G/5G...) để nối ra ngoài; Switch Layer 3 có rất nhiều cổng Ethernet tốc độ cao, tối ưu cho định tuyến **giữa các VLAN nội bộ** trong cùng một tòa nhà/campus chứ không phải kết nối WAN. Trong kiến trúc doanh nghiệp (mục 5.5, Bài 1), Router thường nằm ở vị trí WAN Edge, còn Switch Layer 3 nằm ở lớp Distribution.

## 6. Wireless Access Point (AP)

AP đóng vai trò cầu nối (bridge) giữa mạng có dây (kết nối vào Switch bằng cáp UTP) và mạng không dây (phát sóng Wi-Fi cho client). Về bản chất xử lý frame, AP hoạt động gần giống Bridge/Switch ở Lớp 2 nhưng thêm phần chuyển đổi giữa 2 loại phương tiện truyền dẫn khác nhau (mục 3, [[Bai-03_Phuong-tien-truyen-dan-mang|Bài 3]]). Trong mạng doanh nghiệp có nhiều AP, việc quản lý riêng lẻ từng AP không hiệu quả — đây là lý do tồn tại Wireless LAN Controller (WLC) để quản lý tập trung, nội dung chi tiết ở Bài 24-25.

## 7. Module mở rộng: RJ45, SFP, PoE

### 7.1 Cổng RJ45

Cổng kết nối tiêu chuẩn cho cáp UTP/STP, dùng phổ biến nhất trên switch/router/PC — đã học chi tiết cách bấm ở [[Bai-03_Phuong-tien-truyen-dan-mang|Bài 3]].

### 7.2 SFP, SFP+, QSFP — module quang cắm rời

Là các khe cắm (slot) cho phép gắn module quang tháo lắp được (hot-swappable) thay vì cổng cố định — dùng để kết nối cáp quang giữa các switch, hoặc giữa switch với thiết bị ở khoảng cách xa hơn giới hạn của UTP (100m).

| Loại | Tốc độ tối đa | Ghi chú |
|---|---|---|
| SFP | ~1 Gbps (một số hỗ trợ đến 4.25 Gbps cho Fibre Channel) | Phổ biến cho uplink Gigabit giữa các switch access/distribution |
| SFP+ | 10 Gbps | Cùng kích thước vật lý với SFP nhưng tốc độ cao hơn; **KHÔNG cắm vào cổng chỉ hỗ trợ SFP thường được (không lên link); chiều tương thích ngược là module SFP (1G) cắm vào cổng SFP+ thì chạy ở 1 Gbps, tùy nền tảng** |
| QSFP / QSFP+ / QSFP28 | QSFP gốc 4x1 Gbps; QSFP+ 40 Gbps; QSFP28 100 Gbps | Gấp 4 lần băng thông SFP+ trong cùng không gian vật lý, dùng cho kết nối lớp Core hiệu năng cao |

> [!question] Cắm nhầm SFP+ vào cổng SFP có sao không?
> Thường không hỏng thiết bị, nhưng module SFP+ sẽ không lên link vì cổng chỉ hỗ trợ SFP thường không hỗ trợ 10 Gbps (chiều ngược lại — module SFP 1G cắm vào cổng SFP+ — thường chạy được ở 1 Gbps, tùy nền tảng) — đây là lỗi thực chiến phổ biến khi nâng cấp hạ tầng: mua nhầm module tốc độ cao nhưng cắm vào switch cũ chỉ hỗ trợ SFP thường, dẫn đến "nâng cấp cáp quang nhưng link không lên" mà không rõ nguyên nhân nếu không kiểm tra kỹ specification của cổng.

### 7.3 PoE — Power over Ethernet

Công nghệ truyền điện cùng lúc với dữ liệu qua cùng một sợi cáp UTP — giúp cấp nguồn cho AP, camera IP, điện thoại VoIP mà không cần kéo dây điện riêng (rất hữu ích khi lắp AP trên trần nhà hoặc camera ở vị trí khó kéo điện).

| Chuẩn | Tên gọi | Công suất tại nguồn (PSE) | Công suất thực nhận (PD) | Ứng dụng điển hình |
|---|---|---|---|---|
| 802.3af | PoE | 15.4W | ~12.95W | Điện thoại IP, AP cơ bản |
| 802.3at | PoE+ | 30W | ~25.5W | AP Wi-Fi hiện đại, camera PTZ |
| 802.3bt (Type 3/4) | PoE++ | 60W / 90-100W | ~51W / ~71W | Camera có sưởi/quạt, màn hình LED nhỏ, AP đa băng tần cao cấp |

Nguyên tắc tương thích: thiết bị cấp nguồn (PSE) chuẩn cao hơn **luôn cấp được** cho thiết bị nhận (PD) chuẩn thấp hơn (switch PoE++ cấp được cho AP chỉ cần PoE thường), nhưng chiều ngược lại thì không — thiết bị PD cần nhiều công suất hơn khả năng của PSE sẽ không khởi động được hoặc chạy thiếu ổn định.

## 8. Lab thực hành

**Môi trường**: Cisco Packet Tracer.

1. Dựng 3 PC nối qua 1 Hub, gửi ping đồng thời từ 2 PC — quan sát hiện tượng collision (Packet Tracer mô phỏng bằng packet màu đỏ/hiệu ứng nhấp nháy ở chế độ Simulation).
2. Dựng lại y hệt 3 PC đó nhưng thay Hub bằng Switch, lặp lại thao tác — xác nhận không còn hiện tượng collision.
3. Vào chế độ Simulation, gửi 1 gói ping đầu tiên giữa 2 PC qua Switch, quan sát switch flood ra mọi cổng ở lần đầu (do MAC table còn trống), sau đó gửi lần thứ 2 chỉ đi đúng 1 cổng — kiểm tra bảng MAC bằng lệnh `show mac address-table` trên CLI switch.

**Kết quả kỳ vọng**: thấy rõ bằng thực hành sự khác biệt hiệu năng và hành vi giữa Hub và Switch, đồng thời quan sát trực tiếp quá trình học/flood/forward đã học ở mục 4.1.

## 9. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Switch và Router là 2 tên gọi khác nhau của cùng 1 thiết bị | Nhầm lẫn vì cả 2 đều "chuyển tiếp dữ liệu" | Switch hoạt động trong cùng 1 mạng (Lớp 2, theo MAC); Router kết nối giữa các mạng khác nhau (Lớp 3, theo IP) |
| Switch Layer 3 chính là Router, có thể thay thế hoàn toàn | Cùng làm được định tuyến IP cơ bản | Switch Layer 3 tối ưu cho định tuyến nội bộ giữa nhiều VLAN, thiếu các cổng WAN đa dạng mà Router chuyên dụng có |
| Cứ cắm được vào cổng SFP là chạy đúng tốc độ ghi trên hộp module | Tốc độ thực tế bị giới hạn bởi khả năng của CHÍNH cổng switch, không chỉ module | Module SFP+ cắm vào cổng chỉ hỗ trợ SFP sẽ không lên link; ngược lại module SFP 1G cắm vào cổng SFP+ thường chạy 1 Gbps (tùy nền tảng) |
| PoE chuẩn nào cũng cấp đủ điện cho mọi thiết bị | Nhầm lẫn công suất PSE cấp ra với công suất PD thực sự cần | Phải kiểm tra PD cần bao nhiêu Watt và PSE hỗ trợ chuẩn nào tương ứng, PD cần nhiều hơn PSE cấp được sẽ không hoạt động ổn định |

## 10. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao Switch được xem là bước tiến vượt trội so với Hub?**
→ Hub phát tín hiệu ra mọi cổng, mọi thiết bị chung 1 collision domain, chỉ half-duplex. Switch học địa chỉ MAC theo từng cổng, mỗi cổng là 1 collision domain riêng, chuyển frame có chọn lọc (forward) thay vì phát tràn lan, hỗ trợ full-duplex — tăng hiệu năng và giảm khả năng nghe lén traffic không liên quan.

**2. Trình bày quy trình học địa chỉ MAC của Switch (learn — flood — forward — filter).**
→ Learn: ghi {MAC nguồn : cổng} vào bảng khi nhận frame. Flood: nếu MAC đích chưa có trong bảng, gửi ra mọi cổng trừ cổng nhận. Forward: nếu MAC đích đã học được, chỉ gửi ra đúng 1 cổng. Filter: nếu MAC nguồn và đích cùng nằm trên 1 cổng, không chuyển tiếp.

**3. Switch và Router khác nhau cơ bản ở điểm nào?**
→ Switch hoạt động Lớp 2, chuyển frame theo địa chỉ MAC, chỉ trong phạm vi cùng 1 mạng/broadcast domain. Router hoạt động Lớp 3, định tuyến packet theo địa chỉ IP, kết nối giữa các mạng (subnet) khác nhau.

**4. Vì sao cần SFP/SFP+ thay vì dùng cổng RJ45 thông thường ở mọi kết nối?**
→ RJ45/UTP giới hạn 100m; khi cần kết nối xa hơn (giữa các tòa nhà, giữa switch Core-Distribution ở khoảng cách lớn) hoặc cần băng thông cao hơn (10G/40G), phải dùng module quang cắm qua khe SFP/SFP+/QSFP.

**5. PoE hoạt động ra sao? Vì sao switch PoE++ vẫn cấp nguồn được cho thiết bị chỉ cần chuẩn PoE thường?**
→ PoE truyền điện DC cùng lúc với dữ liệu qua cáp UTP để cấp nguồn cho thiết bị đầu cuối (AP, camera IP, điện thoại VoIP) mà không cần dây điện riêng. Chuẩn cao hơn (PoE++) luôn tương thích ngược, cấp được công suất thấp hơn mà thiết bị PoE/PoE+ yêu cầu — nhưng chiều ngược lại (PSE chuẩn thấp cấp cho PD cần công suất cao) thì không đủ.

## 11. Tài liệu tham khảo

- [Power over Ethernet Standards (802.3af, 802.3at, 802.3bt) — NVT Phybridge](https://www.nvtphybridge.com/power-over-ethernet-standards/)
- [SFP vs SFP+ vs QSFP: What's the difference? — Black Box](https://www.blackbox.com/insights/blackbox-explains/inner/detail/networking/connectivity/sfp-vs-sfp-vs-qsfp-what-s-the-difference)
- [Tenable — Cisco Smart Install: How to Prevent Attacks on Switches](https://www.tenable.com/blog/cisco-smart-install-how-to-prevent-attacks-on-switches)
- [Kaspersky ICS CERT — Attack on Cisco switches](https://ics-cert.kaspersky.com/publications/news/2018/04/11/cisco-attack/)

---
**Liên kết**: [[Bai-03_Phuong-tien-truyen-dan-mang|← Bài 3]] · Bài 5 - Địa chỉ IP Version 4 →
