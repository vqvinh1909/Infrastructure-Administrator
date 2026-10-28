---
title: Bài 25 - Cisco Wireless LAN Controller (WLC)
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — WLC là mô hình quản trị Wifi chuẩn của mọi doanh nghiệp vừa/lớn, hiểu CAPWAP là kiến thức nền cho cả câu hỏi lý thuyết lẫn troubleshooting thực tế"
tags: [networking, wireless, wlc, capwap, cisco, sysadmin]
aliases: [WLC, Wireless Controller, CAPWAP, Lightweight AP]
related: ["[[Bai-24_Wireless-LAN]]"]
last_review: 2026-07-21
---

# Bài 25: Cisco Wireless LAN Controller (WLC)

> [!important] Tại sao cần file này
> [[Bai-24_Wireless-LAN|Bài 24]] đã nói tới mô hình **Lightweight AP + Wireless Controller** như giải pháp chuẩn cho doanh nghiệp vừa/lớn, nhưng chưa đi sâu vào cách nó vận hành. Bài này giải thích chi tiết vì sao mô hình quản trị tập trung lại cần thiết, cơ chế giao thức CAPWAP đứng sau nó, và cách triển khai/cấu hình một WLC thực tế.

## 1. Vì sao cần mô hình quản trị Wifi tập trung

Với mô hình **Autonomous AP** (đã nói ở Bài 24), mỗi AP tự lưu cấu hình riêng — khi doanh nghiệp có hàng chục, hàng trăm AP trải khắp nhiều tòa nhà, việc cấu hình/cập nhật từng AP một trở nên không khả thi (tương tự vấn đề quản lý VLAN thủ công trên nhiều switch mà [[Bai-09_Trunking-DTP-VTP|VTP]] từng giải quyết ở quy mô switch). Ngoài ra, Autonomous AP hoạt động độc lập nên không thể phối hợp để tối ưu Roaming, cân bằng tải Client giữa các AP, hay phát hiện nhiễu/rogue AP trên toàn hệ thống.

> [!question] Mô hình WLC giải quyết vấn đề gì mà VTP (Bài 9) không thể giải quyết được?
> VTP chỉ đồng bộ **thông tin VLAN** giữa các switch trong cùng domain — nó không quản lý cấu hình vận hành hàng ngày như công suất phát sóng, kênh sóng, danh sách Client kết nối, hay chính sách bảo mật của từng AP. WLC đi xa hơn: nó là điểm quản trị **tập trung toàn diện** cho hạ tầng không dây — từ cấu hình SSID, bảo mật, công suất phát, tới giám sát Client theo thời gian thực và tự động điều chỉnh kênh sóng (RRM — Radio Resource Management) để tránh nhiễu giữa các AP lân cận, việc mà VTP không có khái niệm tương đương.

## 2. Môi trường cần thiết trước khi triển khai

- **Kết nối mạng có dây (DS)** ổn định giữa AP và WLC — CAPWAP hoạt động qua hạ tầng Ethernet/IP sẵn có, không thay thế mạng có dây.
- **VLAN riêng cho quản lý AP/WLC** (Management VLAN) — tách biệt traffic quản trị khỏi traffic Client, theo nguyên tắc phân đoạn mạng đã học ở [[Bai-06_VLAN|Bài 6]].
- **DHCP** cấp IP cho AP (đã học ở [[Bai-19_Dich-vu-DHCP|Bài 19]]) — nếu dùng DHCP Option 43, cần cấu hình thêm để AP tự tìm ra IP của WLC trong quá trình discovery (mục 4).
- **NTP** đồng bộ thời gian giữa WLC và AP (đã học ở [[Bai-18_NTP-va-Syslog|Bài 18]]) — vì CAPWAP dùng DTLS, và chứng thực TLS/DTLS nhạy cảm với sai lệch thời gian giữa 2 bên.
- **Đủ băng thông đường trunk giữa AP và WLC** nếu WLC đặt ở vị trí tập trung (ví dụ Data Center) trong khi AP trải khắp nhiều tòa nhà — vì toàn bộ traffic Client trong mô hình Split MAC phải tunnel qua CAPWAP về WLC (mục 3).

## 3. Kiến trúc CAPWAP — Split MAC

**CAPWAP** (Control And Provisioning of Wireless Access Points — RFC 5415) là giao thức chuẩn hóa việc AP giao tiếp với WLC, kế thừa từ LWAPP (giao thức độc quyền Cisco trước đó). CAPWAP chia trách nhiệm của lớp MAC 802.11 theo mô hình **Split MAC**:

| Chức năng | Xử lý tại |
|---|---|
| Các tác vụ nhạy cảm thời gian thực (beacon, phản hồi probe, mã hóa/giải mã khung 802.11) | **AP** (tại chỗ, để đảm bảo độ trễ thấp) |
| Chính sách bảo mật, xác thực, quản lý cấu hình, điều phối RF trên toàn hệ thống (RRM) | **WLC** (tập trung) |
| Toàn bộ dữ liệu và khung quản lý của Client | Đóng gói qua **đường hầm CAPWAP**, chuyển về WLC xử lý |

```
AP (tai cho: beacon, ma hoa 802.11)
   |
   |  CAPWAP tunnel (UDP 5246 Control + UDP 5247 Data, ma hoa DTLS)
   v
WLC (tap trung: chinh sach, xac thuc, RRM)
```

Hai kênh CAPWAP tách biệt: **kênh Control** (UDP 5246, mã hóa DTLS mặc định — mang lệnh cấu hình, quản lý) và **kênh Data** (UDP 5247 — mang traffic Client thực tế, DTLS cho kênh này mặc định tắt và cần bật riêng nếu muốn mã hóa toàn bộ traffic Client).

> [!question] Vì sao CAPWAP vẫn để AP tự xử lý phần mã hóa/giải mã 802.11 tại chỗ thay vì tập trung hết về WLC?
> Vì các tác vụ ở lớp MAC 802.11 (như trả lời beacon, xác nhận khung) có yêu cầu **độ trễ cực thấp** — nếu mọi khung phải đi qua mạng về WLC rồi chờ phản hồi mới xử lý tiếp, độ trễ tăng đáng kể có thể phá vỡ giao thức 802.11 (vốn có các mốc thời gian chờ phản hồi rất ngắn ở tầng vật lý/liên kết). Split MAC là thỏa hiệp hợp lý: giữ các tác vụ **nhạy cảm thời gian thực** ở gần Client nhất (tại AP), trong khi tập trung các tác vụ **không nhạy cảm thời gian thực** (chính sách, xác thực, quản lý) về WLC để dễ quản trị.

## 4. Quá trình AP gia nhập (Join) WLC

AP lightweight trải qua 2 giai đoạn khi khởi động: **Discovery** rồi **Join**.

**Giai đoạn Discovery** — AP tìm WLC bằng nhiều cách (thử theo thứ tự cho tới khi tìm được):
1. Gửi broadcast Layer 3 trên subnet cục bộ để tìm WLC.
2. Dùng địa chỉ IP của các WLC đã lưu từ lần kết nối trước.
3. Dùng **DHCP Option 43** — server DHCP trả về địa chỉ IP của WLC kèm theo thông tin IP cấp cho AP.
4. Phân giải DNS tên `CISCO-CAPWAP-CONTROLLER.<local-domain>` để tìm IP của WLC.

**Giai đoạn Join** — sau khi có địa chỉ WLC, AP gửi Join Request kèm **chứng chỉ số (certificate)** để xác thực; nếu hợp lệ, WLC gửi Join Response. Sau đó:
1. AP so sánh phiên bản phần mềm (image) của mình với WLC — nếu khác, AP **tải image mới từ WLC và khởi động lại** để đồng bộ phiên bản.
2. Khi image khớp, AP tải **cấu hình** từ WLC và chuyển sang trạng thái **Registered** — chính thức hoạt động dưới sự quản lý của WLC.

> [!question] Vì sao AP bắt buộc phải đồng bộ đúng phiên bản phần mềm với WLC trước khi được cấp cấu hình?
> Vì CAPWAP là giao thức khá nhạy với sự tương thích giữa phiên bản phần mềm AP và WLC — các trường dữ liệu, tính năng RF, hay cơ chế bảo mật có thể khác nhau giữa các phiên bản. Nếu để AP chạy phiên bản không khớp mà vẫn nhận cấu hình, nguy cơ AP hiểu sai lệnh cấu hình hoặc thiếu tính năng WLC đang áp dụng là rất cao. Bắt buộc đồng bộ image trước đảm bảo toàn bộ AP trong hệ thống vận hành nhất quán, giống tinh thần "đồng bộ trạng thái trước khi tin tưởng dữ liệu" đã gặp ở nhiều giao thức khác trong khóa học (ví dụ điều kiện Neighbor của OSPF/EIGRP).

## 5. Cấu hình cơ bản triển khai WLC

Ví dụ minh họa các bước cấu hình chính trên CLI của WLC (mô hình AireOS; Catalyst 9800 chạy IOS-XE nên dùng cú pháp khác: wlan profile, policy profile, tag):

> **Lưu ý cập nhật:** AireOS là nền tảng legacy — WLC 3504 đã hết bán (EoS) 31/01/2021 và hết hỗ trợ 31/01/2027; AireOS 8.10 đã có thông báo EoL, Cisco khuyến nghị chuyển sang Catalyst 9800.

```
(WLC) > config interface address management <IP> <mask> <gateway>
(WLC) > config wlan create 1 CONGTY-WIFI CONGTY-WIFI
(WLC) > config wlan security wpa wpa2 enable 1
(WLC) > config wlan security wpa akm psk enable 1
(WLC) > config wlan security wpa akm psk set-key ascii Congty@2026 1
(WLC) > config wlan interface 1 management
(WLC) > config wlan enable 1
```

Nếu dùng DHCP Option 43 để AP tự tìm WLC, cấu hình phía DHCP Server (IOS) ví dụ (byte thứ hai của giá trị là độ dài = 4 × số WLC: 1 WLC dùng `f104`, ví dụ `f104c0a8010a` cho WLC 192.168.1.10; `f108` chỉ đúng khi có 2 WLC):

```
Router(config)# ip dhcp pool WIFI-AP-POOL
Router(dhcp-config)# option 43 hex f104<IP-WLC-dang-hex>
```

## 6. Verify & Troubleshoot

```
(WLC) > show ap summary
(WLC) > show ap join stats summary <AP-name>
(WLC) > show client summary
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| AP không xuất hiện trong `show ap summary` | AP chưa qua được giai đoạn Discovery — sai DHCP Option 43, hoặc AP không tới được WLC qua Layer 3 | Kiểm tra cấu hình Option 43, xác nhận route/firewall cho phép UDP 5246/5247 giữa AP và WLC |
| AP bị kẹt ở trạng thái "Downloading image" liên tục | Lỗi trong quá trình tải image (đường truyền không ổn định, dung lượng lưu trữ AP không đủ) | Kiểm tra kết nối, dung lượng flash trên AP, thử reset AP về factory rồi join lại |
| AP join được nhưng Client không kết nối được | Sai cấu hình WLAN (SSID/bảo mật) trên WLC, hoặc VLAN interface gán cho WLAN không đúng | Kiểm tra `config wlan interface`, xác nhận VLAN đã tồn tại và có DHCP phục vụ đúng subnet |
| Chứng chỉ AP bị từ chối khi Join dù Discovery thành công | Sai lệch thời gian giữa AP và WLC khiến DTLS/certificate không hợp lệ | Kiểm tra đồng bộ NTP giữa AP và WLC (Bài 18) |

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer (hỗ trợ WLC 2504/3504 và AP lightweight). Lưu ý các WLC này chạy AireOS — nền tảng legacy (xem lưu ý cập nhật ở mục 5).

1. Đặt 1 WLC và 2 AP lightweight trong topology, kết nối qua switch (đóng vai trò DS).
2. Cấu hình Management Interface trên WLC, xác nhận cả 2 AP tự động Discovery và Join thành công qua `show ap summary`.
3. Tạo 1 WLAN mới với SSID + bảo mật WPA2-PSK, gán vào 1 VLAN Client cụ thể.
4. Kết nối 1 thiết bị không dây, xác nhận nhận đúng IP và thấy Client trong `show client summary` trên WLC.

**Kết quả kỳ vọng**: hiểu luồng Discovery → Join → Registered của AP trong thực tế, và thấy rõ vai trò quản trị tập trung của WLC so với Autonomous AP ở Bài 24.

## 8. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| WLC thay thế hoàn toàn switch/router trong mạng có dây | Nhầm lẫn vai trò WLC với hạ tầng có dây | WLC chỉ quản lý phần **Wifi**, vẫn cần switch/router làm DS kết nối AP-WLC như bình thường (mục 2) |
| CAPWAP mã hóa toàn bộ traffic Client mặc định | Bỏ qua khác biệt giữa kênh Control và Data | Kênh Control (UDP 5246) mã hóa DTLS mặc định, nhưng kênh Data (UDP 5247, mang traffic Client) mặc định **không** mã hóa, cần bật riêng nếu cần (mục 3) |
| AP lightweight có thể hoạt động độc lập nếu mất kết nối tới WLC | Bỏ qua bản chất phụ thuộc của mô hình Split MAC | Tùy chế độ cấu hình, phần lớn AP lightweight sẽ mất chức năng phục vụ Client mới hoặc hoạt động hạn chế nếu mất liên lạc với WLC — không hoạt động độc lập như Autonomous AP |
| Chỉ cần đúng IP WLC là AP join được ngay | Bỏ qua yêu cầu về phiên bản phần mềm và chứng chỉ | AP còn phải qua bước xác thực certificate và đồng bộ đúng image trước khi được cấp cấu hình (mục 4) |

## 9. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. CAPWAP là gì và giải quyết vấn đề gì?**
→ CAPWAP (Control And Provisioning of Wireless Access Points) là giao thức chuẩn hóa (RFC 5415) cho phép AP lightweight giao tiếp và được quản lý tập trung bởi WLC, giải quyết vấn đề quản trị phân tán, khó đồng bộ chính sách và khó tối ưu RF khi số lượng AP lớn.

**2. Phân biệt vai trò kênh Control và kênh Data trong CAPWAP.**
→ Kênh Control (UDP 5246) mang lệnh cấu hình/quản lý giữa AP và WLC, mã hóa DTLS mặc định. Kênh Data (UDP 5247) mang traffic Client thực tế được tunnel về WLC, mặc định không mã hóa DTLS trừ khi được bật riêng.

**3. Giải thích ngắn gọn 2 giai đoạn Discovery và Join của AP.**
→ Discovery: AP tìm địa chỉ IP của WLC (qua broadcast, IP đã lưu, DHCP Option 43, hoặc DNS). Join: AP gửi Join Request kèm certificate, nếu hợp lệ thì đồng bộ image phần mềm với WLC (tải lại nếu khác), sau đó tải cấu hình và chuyển sang trạng thái Registered.

**4. Vì sao mô hình Split MAC lại giữ một số chức năng ở AP thay vì tập trung hết về WLC?**
→ Vì các chức năng nhạy cảm thời gian thực của lớp MAC 802.11 (như trả lời beacon/probe, mã hóa khung) cần độ trễ cực thấp — để tại AP tránh được độ trễ mạng khi phải gửi ngược về WLC xử lý, trong khi các chức năng không nhạy cảm thời gian thực (chính sách, xác thực, RRM) tập trung về WLC để dễ quản trị.

**5. Nếu 1 AP bị từ chối khi Join dù đã Discovery thành công, hướng kiểm tra đầu tiên là gì?**
→ Kiểm tra đồng bộ thời gian (NTP) giữa AP và WLC, vì DTLS/certificate rất nhạy với sai lệch thời gian; sau đó kiểm tra chứng chỉ số của AP có hợp lệ/còn hạn hay không.

## 10. Tài liệu tham khảo

- [Cisco — Control and Provisioning of Wireless Access Points (CAPWAP) Overview](https://www.cisco.com/c/en/us/td/docs/wireless/outdoor_industrial/iw9165d/software/configuration/261x/b-iw9165d-scg-261x/g-control-and-provisioning-of-wireless-access-points/c-capwap-overview.html)
- [Cisco — Understand the AP Join Process with the Catalyst 9800 WLC](https://www.cisco.com/c/en/us/support/docs/wireless/catalyst-9120axe-access-point/221056-understand-the-ap-join-process-with-the.html)
- [Cisco — Troubleshoot a Lightweight AP that Fails to Join a WLC](https://www.cisco.com/c/en/us/support/docs/wireless/5500-series-wireless-controllers/119286-lap-notjoin-wlc-tshoot.html)
- [Wikipedia — Control and Provisioning of Wireless Access Points protocol](https://en.wikipedia.org/wiki/Control_and_Provisioning_of_Wireless_Access_Points_protocol)

---
**Liên kết**: [[Bai-24_Wireless-LAN|← Bài 24]] · [[Bai-26_Bao-mat-Lop-2|Bài 26 - Bảo mật Lớp 2 →]]
