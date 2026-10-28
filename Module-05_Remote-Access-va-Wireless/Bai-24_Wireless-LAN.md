---
title: Bài 24 - Wireless LAN
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — Wireless LAN là hạ tầng bắt buộc trong mọi văn phòng hiện đại, và phân biệt đúng chuẩn bảo mật WPA2/WPA3 là câu hỏi phỏng vấn rất phổ biến"
tags: [networking, wireless, wifi, 802.11, wpa2, wpa3, sysadmin]
aliases: [WLAN, Wifi, 802.11, BSS, ESS, WPA3]
related: ["[[Bai-25_Cisco-Wireless-Controller]]", "[[Bai-03_Phuong-tien-truyen-dan-mang]]"]
last_review: 2026-07-21
---

# Bài 24: Wireless LAN

> [!important] Tại sao cần file này
> [[Bai-03_Phuong-tien-truyen-dan-mang|Bài 3]] đã giới thiệu Wifi như 1 trong các phương tiện truyền dẫn, nhưng chưa đi sâu vào kiến trúc và vận hành thực tế. Từ Bài 7 tới Bài 23, toàn bộ hạ tầng học được đều là **mạng có dây** — trong khi thực tế văn phòng hiện đại, phần lớn thiết bị đầu cuối (laptop, điện thoại, máy in không dây) kết nối qua **Wireless LAN**. Bài này xây dựng nền tảng kiến trúc 802.11 trước khi học quản trị tập trung qua Wireless Controller ở [[Bai-25_Cisco-Wireless-Controller|Bài 25]].

## 1. Mạng 802.11 — Mô hình BSS, IBSS, ESS, DS

| Thuật ngữ | Ý nghĩa |
|---|---|
| **BSS** (Basic Service Set) | 1 Access Point (AP) cùng toàn bộ Client đang kết nối vào nó — đơn vị cơ bản nhất của mạng Wifi |
| **IBSS** (Independent BSS) | Mô hình **Ad-hoc**, các thiết bị kết nối trực tiếp ngang hàng (peer-to-peer) với nhau, **không cần AP** — ít dùng trong doanh nghiệp, chỉ phù hợp nhóm nhỏ (dưới 10 thiết bị) |
| **DS** (Distribution System) | Hạ tầng có dây (thường là Ethernet) kết nối nhiều AP với nhau, cho phép Client di chuyển giữa các AP mà vẫn ở chung 1 mạng logic |
| **ESS** (Extended Service Set) | Tập hợp nhiều BSS được kết nối qua DS, dùng chung **1 SSID** — đây là mô hình triển khai Wifi doanh nghiệp phổ biến nhất, cho phép người dùng di chuyển giữa các AP mà không bị gián đoạn kết nối |

```
ESS = BSS-1 + BSS-2, ket noi qua DS (cung 1 SSID):
+------------+        +------------+
|    AP-1    |        |    AP-2    |
|   BSS-1    |        |   BSS-2    |
+------------+        +------------+
       |                     |
       +---------------------+
                  |
         +----------------+
         |     SWITCH     |
         | (Distribution  |
         |    System)     |
         +----------------+
```

> [!question] Vì sao mô hình ESS (nhiều AP cùng SSID) lại quan trọng hơn nhiều so với việc chỉ dùng 1 AP đơn lẻ mạnh?
> Vì phạm vi phủ sóng của 1 AP luôn có giới hạn vật lý — 1 tòa nhà nhiều tầng hoặc diện tích lớn không thể phủ sóng đủ chỉ bằng 1 AP dù mạnh tới đâu. ESS cho phép ghép nhiều AP (mỗi AP tạo 1 BSS riêng) cùng chia sẻ 1 SSID, và quan trọng hơn: cho phép cơ chế **Roaming** (mục 3) — Client di chuyển từ vùng phủ sóng AP này sang AP khác mà không cần đăng nhập lại hay gián đoạn kết nối, miễn là cả 2 AP cùng thuộc 1 ESS.

## 2. Các chuẩn 802.11 — băng thông và tần số

| Chuẩn | Tên thương mại | Băng tần | Tốc độ tối đa lý thuyết |
|---|---|---|---|
| 802.11a | — | 5 GHz | 54 Mbps |
| 802.11b | — | 2.4 GHz | 11 Mbps |
| 802.11g | — | 2.4 GHz | 54 Mbps |
| 802.11n | Wifi 4 | 2.4 GHz và 5 GHz (dual-band) | 600 Mbps (với MIMO) |
| 802.11ac | Wifi 5 | 5 GHz | Trên 1 Gbps (với MU-MIMO) |
| 802.11ax | Wifi 6 | 2.4 GHz và 5 GHz (Wifi 6E mở rộng thêm 6 GHz) | Tới 10 Gbps (lý thuyết, với OFDMA) |
| 802.11be | Wifi 7 | 2.4, 5 và 6 GHz (băng 6 GHz yêu cầu WPA3 hoặc OWE; Wi-Fi Alliance chứng nhận từ đầu 2024) | Khoảng 46 Gbps (lý thuyết, kênh 320 MHz) |

> [!question] Vì sao băng tần 5 GHz thường cho tốc độ cao hơn nhưng phạm vi phủ sóng lại kém hơn 2.4 GHz?
> Đây là đánh đổi vật lý cơ bản của sóng vô tuyến: tần số càng cao, băng thông khả dụng càng lớn (cho phép tốc độ truyền dữ liệu cao hơn), nhưng đồng thời sóng tần số cao bị suy hao nhanh hơn qua khoảng cách và vật cản (tường, sàn nhà) — đây là quy luật vật lý chung, tương tự lý do tần số cao dùng cho 5G/mmWave có phạm vi phủ ngắn hơn nhiều so với tần số thấp. Đây là lý do thiết kế Wifi doanh nghiệp thường ưu tiên 5 GHz cho khu vực cần tốc độ cao, mật độ thiết bị dày (ít nhiễu vì ít thiết bị dùng chung băng tần này hơn 2.4 GHz), còn 2.4 GHz dùng bổ sung cho khu vực cần phủ sóng xa hơn.

## 3. SSID, Wifi Repeater, Roaming, Mesh

- **SSID** (Service Set Identifier): tên mạng Wifi hiển thị cho người dùng chọn kết nối — nhiều AP trong cùng 1 ESS (mục 1) cấu hình chung 1 SSID để tạo cảm giác "1 mạng Wifi duy nhất" dù thực chất có nhiều AP vật lý.
- **Wifi Repeater**: thiết bị khuếch đại/lặp lại tín hiệu Wifi hiện có để mở rộng phạm vi phủ sóng — giải pháp đơn giản, chi phí thấp nhưng làm **giảm băng thông thực tế** (vì Repeater phải vừa nhận vừa gửi lại trên cùng kênh sóng), không phù hợp triển khai doanh nghiệp quy mô lớn.
- **Roaming**: khả năng Client tự động chuyển vùng phủ sóng giữa các AP trong cùng ESS mà không bị gián đoạn kết nối (không cần đăng nhập lại) — chất lượng roaming phụ thuộc vào việc các AP có được quản lý tập trung hay không (chi tiết ở [[Bai-25_Cisco-Wireless-Controller|Bài 25]]).
- **Mesh**: kiến trúc các AP tự kết nối với nhau qua **sóng vô tuyến** (thay vì dây Ethernet như DS truyền thống) để mở rộng vùng phủ, mỗi AP Mesh vừa phục vụ Client vừa chuyển tiếp traffic cho AP Mesh khác — phù hợp nơi khó kéo cáp Ethernet tới từng AP.

> [!question] Vì sao Mesh được xem là giải pháp tốt hơn Repeater dù cả 2 đều nhằm mở rộng vùng phủ sóng?
> Vì Repeater chỉ đơn thuần "khuếch đại lại" tín hiệu trên **cùng 1 kênh**, khiến băng thông khả dụng bị chia đôi mỗi lần tín hiệu đi qua 1 Repeater (nhận và gửi lại cùng lúc trên cùng tần số gây tự nhiễu). Mesh hiện đại thường dùng **nhiều kênh/băng tần khác nhau** cho việc kết nối giữa các AP Mesh với nhau (backhaul) và việc phục vụ Client, tránh được vấn đề tự chia băng thông của Repeater, đồng thời có khả năng tự động chọn đường tối ưu giữa các AP Mesh (tương tự tinh thần định tuyến động đã học ở Module 3) nếu 1 AP Mesh gặp sự cố.

## 4. Chuẩn bảo mật Wifi: WEP, WPA, WPA2, WPA3

| Chuẩn | Năm | Mã hóa | Đánh giá |
|---|---|---|---|
| **WEP** | 1997 | Khóa tĩnh 64/128-bit, thuật toán RC4 | **Đã bị phá vỡ hoàn toàn**, không nên dùng trong bất kỳ trường hợp nào hiện nay |
| **WPA** | 2003 | TKIP (cải tiến tạm thời từ WEP) | Giải pháp quá độ, cũng đã lỗi thời |
| **WPA2** | 2004 | AES-CCMP | Tiêu chuẩn phổ biến nhất hiện nay, vẫn đủ an toàn nếu dùng mật khẩu đủ mạnh |
| **WPA3** | 2018 | AES-CCMP 128-bit (bắt buộc với WPA3-Personal; GCMP-256 chỉ dùng cho WPA3-Enterprise 192-bit), xác thực bằng **SAE** (Simultaneous Authentication of Equals) | Chuẩn mới nhất, chống được tấn công dò mật khẩu ngoại tuyến (offline dictionary attack) hiệu quả hơn WPA2 |

> [!question] SAE trong WPA3 giải quyết điểm yếu bảo mật nào mà WPA2 gặp phải?
> WPA2 dùng cơ chế bắt tay **4-way handshake** — kẻ tấn công có thể thu thập gói tin handshake này rồi đem về **dò mật khẩu ngoại tuyến (offline)** bằng cách thử hàng loạt mật khẩu khả dĩ mà không cần tiếp tục tương tác với mạng Wifi thật (không bị giới hạn tốc độ thử). SAE của WPA3 thiết kế lại cơ chế bắt tay sao cho **mỗi lần thử sai đều phải tương tác trực tiếp với AP** — kẻ tấn công không thể "mang gói tin về nhà" để dò offline với tốc độ nhanh tùy ý, khiến tấn công dò mật khẩu trở nên chậm và khó khả thi hơn nhiều so với WPA2.

## 5. Mô hình triển khai cho doanh nghiệp

Doanh nghiệp thường triển khai theo 2 mô hình chính: **Autonomous AP** (mỗi AP tự quản lý cấu hình độc lập, phù hợp quy mô rất nhỏ) và **Lightweight AP + Wireless Controller** (AP chỉ xử lý phần vô tuyến, mọi cấu hình/quản lý tập trung tại 1 Controller — mô hình chuẩn cho doanh nghiệp từ vừa trở lên, chi tiết đầy đủ ở [[Bai-25_Cisco-Wireless-Controller|Bài 25]]). Thiết kế vùng phủ sóng thực tế cần khảo sát site (site survey) để xác định vị trí đặt AP tối ưu, tránh chồng lấn kênh sóng gây nhiễu giữa các AP lân cận.

## 6. Cấu hình cơ bản cho mô hình cụ thể (Autonomous AP)

```
AP(config)# dot11 ssid CONGTY-WIFI
AP(config-ssid)# authentication open
AP(config-ssid)# authentication key-management wpa version 2
AP(config-ssid)# wpa-psk ascii Congty@2026
AP(config)# interface dot11radio 0
AP(config-if)# ssid CONGTY-WIFI
AP(config-if)# no shutdown
```

## 7. Verify & Troubleshoot

```
AP# show dot11 associations
AP# show interfaces dot11radio 0
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Client thấy SSID nhưng không kết nối được | Sai mật khẩu WPA/WPA2, hoặc chuẩn bảo mật Client hỗ trợ không khớp AP (ví dụ Client cũ chỉ hỗ trợ WPA2 nhưng AP chỉ bật WPA3) | Kiểm tra lại `wpa-psk`, cân nhắc bật cả WPA2/WPA3 song song (transition mode) nếu còn thiết bị cũ |
| Tốc độ Wifi chậm dù tín hiệu đầy vạch | Nhiễu do chồng lấn kênh với AP lân cận, hoặc đang dùng băng tần 2.4 GHz đông thiết bị | Kiểm tra kênh sóng đang dùng, đổi sang kênh ít nhiễu hơn hoặc ưu tiên 5 GHz nếu thiết bị hỗ trợ |
| Client bị rớt kết nối khi di chuyển giữa các khu vực trong tòa nhà | Roaming không mượt do các AP không được quản lý tập trung (Autonomous AP hoạt động độc lập) | Cân nhắc chuyển sang mô hình Lightweight AP + Controller (Bài 25) để có Fast Roaming |
| Phạm vi phủ sóng không đủ dù đã thêm Repeater | Repeater làm giảm băng thông và có giới hạn số lần "nối tiếp" hợp lý (mục 3) | Cân nhắc thêm AP thật (qua DS hoặc Mesh) thay vì tiếp tục xếp chồng nhiều Repeater |

## 8. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 2 Access Point, 2 PC/thiết bị không dây.

1. Cấu hình 2 AP cùng SSID theo mục 6, đặt vị trí vùng phủ sóng chồng lấn nhẹ trong Packet Tracer.
2. Kết nối 1 thiết bị không dây, xác nhận nhận đúng IP (qua DHCP, đã học ở [[Bai-19_Dich-vu-DHCP|Bài 19]]) và truy cập được mạng nội bộ.
3. Di chuyển thiết bị từ vùng phủ AP-1 sang AP-2 (thay đổi vị trí trong Packet Tracer), quan sát hành vi kết nối có bị gián đoạn hay chuyển tiếp mượt (Roaming).
4. So sánh cấu hình bảo mật WPA2 vs mở (open, không mật khẩu), xác nhận thiết bị không đúng mật khẩu không kết nối được khi dùng WPA2.

**Kết quả kỳ vọng**: hiểu kiến trúc BSS/ESS/DS thông qua thực hành trực tiếp, và tự tay trải nghiệm hành vi Roaming giữa nhiều AP.

## 9. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Băng tần 5 GHz luôn tốt hơn 2.4 GHz trong mọi trường hợp | Bỏ qua đánh đổi giữa tốc độ và phạm vi phủ sóng | 5 GHz nhanh hơn nhưng phủ sóng kém hơn qua vật cản; 2.4 GHz phù hợp hơn khi cần phủ xa (mục 2) |
| Wifi Repeater là giải pháp mở rộng vùng phủ tương đương với thêm AP thật | Bỏ qua việc Repeater làm giảm băng thông đáng kể | Repeater chỉ nên dùng tạm thời/quy mô nhỏ; doanh nghiệp nên dùng thêm AP qua DS hoặc Mesh (mục 3) |
| WPA2 đã hoàn toàn lỗi thời, phải luôn dùng WPA3 | Đánh giá quá cao mức độ nguy hiểm nếu dùng đúng cách | WPA2 vẫn đủ an toàn nếu dùng mật khẩu đủ mạnh; WPA3 chỉ thực sự cần thiết khi cần chống offline dictionary attack hiệu quả hơn (mục 4) |
| ESS chỉ đơn giản là "nhiều AP cùng tên Wifi" | Bỏ qua vai trò của Distribution System kết nối các BSS | ESS yêu cầu các BSS được kết nối qua DS (thường là mạng có dây) để đảm bảo Client thực sự roaming được giữa các AP, không chỉ là trùng tên SSID ngẫu nhiên |

## 10. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Phân biệt BSS, IBSS, và ESS.**
→ BSS là 1 AP cùng các Client kết nối vào nó. IBSS là mô hình ad-hoc, các thiết bị kết nối trực tiếp ngang hàng không cần AP. ESS là tập hợp nhiều BSS kết nối qua Distribution System, dùng chung 1 SSID, cho phép Client roaming giữa các AP.

**2. Vì sao 802.11ac chỉ hoạt động ở băng tần 5 GHz, không như 802.11n hỗ trợ cả 2.4 GHz và 5 GHz?**
→ 802.11ac được thiết kế tập trung vào tốc độ cao (MU-MIMO, kênh rộng hơn), và các yêu cầu kỹ thuật này phù hợp hơn với đặc tính băng thông rộng của 5 GHz — 2.4 GHz có băng thông hẹp hơn, không đáp ứng đủ yêu cầu thiết kế của chuẩn ac.

**3. SAE trong WPA3 cải tiến gì so với 4-way handshake của WPA2?**
→ SAE thiết kế lại cơ chế bắt tay sao cho mỗi lần thử mật khẩu sai đều cần tương tác trực tiếp với AP, ngăn kẻ tấn công thu thập gói tin để dò mật khẩu ngoại tuyến (offline) như có thể làm với 4-way handshake của WPA2.

**4. Vì sao Roaming mượt mà hơn khi dùng mô hình Lightweight AP + Controller so với Autonomous AP?**
→ Vì các AP trong mô hình Controller được quản lý tập trung, chia sẻ thông tin về Client đang kết nối và có thể phối hợp quyết định chuyển giao (handoff) nhanh hơn — trong khi Autonomous AP hoạt động độc lập, không có cơ chế phối hợp trực tiếp giữa các AP để tối ưu quá trình chuyển vùng.

**5. Vì sao Wifi Repeater không phù hợp cho triển khai Wifi doanh nghiệp quy mô lớn?**
→ Vì Repeater phải vừa nhận vừa gửi lại tín hiệu trên cùng kênh sóng, làm giảm đáng kể băng thông thực tế khả dụng mỗi lần tín hiệu đi qua 1 Repeater — không phù hợp khi cần đảm bảo hiệu năng ổn định cho nhiều người dùng đồng thời.

## 11. Tài liệu tham khảo

- [Cisco Press — Wireless Topologies](https://www.ciscopress.com/articles/article.asp?p=2999384&seqNum=4)
- [Data-Alliance — WiFi Network Standards Compared](https://www.data-alliance.net/blog/wifi-network-standards-80211ac-compared-to-80211n-80211g-80211a-and-80211b)
- [StationX — WPA2 vs WPA3 Comparison](https://www.stationx.net/wpa2-vs-wpa3/)

---
**Liên kết**: [[Bai-23_Network-Load-Balancing|← Bài 23]] · [[Bai-25_Cisco-Wireless-Controller|Bài 25 - Cisco Wireless Controller →]]
