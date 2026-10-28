---
title: Bài 3 - Phương tiện truyền dẫn mạng
type: Foundation
difficulty: Beginner
lab_required: true
exam: "Sysadmin Junior — nền tảng Layer 1 bắt buộc trước khi bấm cáp/lắp đặt thiết bị thật"
tags: [networking, fundamentals, cabling, utp, fiber, wireless, physical-layer, sysadmin]
aliases: [Network Media, Phuong tien truyen dan]
related: ["[[Bai-02_Mo-hinh-OSI-TCPIP]]", "[[Bai-04_Thiet-bi-ket-noi-mang]]"]
last_review: 2026-07-21
---

# Bài 3: Phương tiện truyền dẫn mạng

> [!important] Tại sao cần file này
> Đây là nội dung cụ thể hóa của Lớp 1 (Physical) đã học ở [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]. Trong thực tế vận hành, một tỷ lệ rất lớn sự cố mạng "khó hiểu" (kết nối chập chờn, tốc độ tụt bất thường, packet loss không rõ nguyên nhân) bắt nguồn từ chính lớp thấp nhất này — sai loại cáp so với khoảng cách, cáp chạy sát nguồn nhiễu, hoặc bấm sai chuẩn. Hiểu đúng đặc tính từng loại phương tiện truyền dẫn giúp loại trừ nhanh nguyên nhân gốc thay vì đoán mò ở các lớp cao hơn.

## 1. Phương tiện truyền dẫn là gì

Phương tiện truyền dẫn (network media) là môi trường vật lý mang tín hiệu từ thiết bị này sang thiết bị khác — tương ứng đúng với Lớp 1 (Physical) trong mô hình OSI. Chia làm 2 nhóm lớn: **có dây** (tín hiệu điện qua kim loại hoặc ánh sáng qua sợi quang) và **không dây** (sóng điện từ qua không khí). Lựa chọn sai loại phương tiện cho một tình huống cụ thể — ví dụ dùng UTP thường cho khoảng cách 200m, hoặc dùng Wi-Fi cho kết nối cần độ ổn định tuyệt đối giữa 2 tòa nhà — là nguyên nhân gốc rễ của rất nhiều sự cố "chập chờn" khó chẩn đoán.

## 2. Phương tiện có dây

### 2.1 Coaxial Cable (Cáp đồng trục)

Cấu tạo gồm 1 lõi đồng trung tâm, lớp cách điện, lớp lưới kim loại chống nhiễu, và vỏ bọc ngoài — dùng chung một dây dẫn cho tín hiệu. Từng là chuẩn phổ biến của Ethernet đời đầu (10BASE5/10BASE2) nhưng đã bị UTP thay thế hoàn toàn trong hạ tầng LAN doanh nghiệp vì khó thi công, khó mở rộng theo mô hình Star (đặc tính vốn phù hợp với topology Bus — xem lại [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]]). Ngày nay coaxial chỉ còn xuất hiện ở đường truyền cáp quang/truyền hình cáp (CATV) chặng cuối tới nhà dân (cable modem/DOCSIS) — không còn dùng để nối PC/switch trong LAN.

### 2.2 UTP — Unshielded Twisted Pair (Cáp xoắn đôi không vỏ chống nhiễu)

Cấu tạo gồm 4 cặp dây đồng xoắn vào nhau. Việc xoắn đôi không phải ngẫu nhiên: hai dây trong cùng một cặp mang tín hiệu ngược pha, khi bị nhiễu từ bên ngoài tác động, nhiễu đó tác động gần như bằng nhau lên cả 2 dây và bị triệt tiêu ở đầu nhận (cancellation) — đây là cơ chế chống nhiễu bị động của twisted pair, không cần lớp chống nhiễu vật lý bổ sung.

| Chuẩn | Băng thông | Tốc độ tối đa | Khoảng cách tối đa | Ghi chú |
|---|---|---|---|---|
| Cat5e | 100 MHz | 1 Gbps | 100m | Phổ biến trong hạ tầng cũ, vẫn đáp ứng đa số nhu cầu văn phòng |
| Cat6 | 250 MHz | 10 Gbps (37-55m) / 1 Gbps (100m) | 100m (giảm tốc độ nếu chạy 10G) | Chuẩn phổ biến cho lắp đặt mới hiện nay |
| Cat6a | 500 MHz | 10 Gbps | 100m (giữ nguyên tốc độ full 100m) | Có cả bản UTP lẫn bản có lớp chống nhiễu (F/UTP, S/FTP); bản shielded phù hợp môi trường nhiều thiết bị điện |
| Cat7 | 600 MHz | 10 Gbps+ | 100m | Chống nhiễu tốt nhất nhưng **không phải chuẩn TIA/EIA chính thức** ở Bắc Mỹ, dùng đầu nối riêng — hiếm dùng trong hạ tầng doanh nghiệp phổ thông |

### 2.3 STP — Shielded Twisted Pair (Cáp xoắn đôi có vỏ chống nhiễu)

Giống UTP về cấu tạo lõi (4 cặp xoắn) nhưng bổ sung lớp vỏ kim loại chống nhiễu quanh mỗi cặp hoặc quanh toàn bộ cáp (tùy loại: STP, FTP, SFTP). Dùng trong môi trường có nhiều nguồn nhiễu điện từ mạnh — nhà máy, phòng máy có động cơ/máy biến áp lớn, hoặc nơi cáp mạng buộc phải chạy song song gần đường điện. Đánh đổi: giá cao hơn, cứng hơn, khó thi công hơn UTP, và **lớp chống nhiễu bắt buộc phải được tiếp đất (grounding) đúng cách** — nếu không, lớp shield có thể phản tác dụng, biến thành ăng-ten thu nhiễu.

### 2.4 Fibre Optic (Cáp quang)

Truyền dữ liệu bằng xung ánh sáng thay vì tín hiệu điện, nên hoàn toàn miễn nhiễm với nhiễu điện từ (EMI/RFI) — ưu điểm lớn nhất so với mọi loại cáp đồng.

| | Single-mode (SM) | Multi-mode (MM) |
|---|---|---|
| Đường kính lõi | Nhỏ (8-10 µm) — chỉ 1 tia sáng truyền thẳng | Lớn (50-62.5 µm) — nhiều tia sáng truyền theo nhiều góc khác nhau |
| Khoảng cách | Hàng chục đến hàng trăm km | Vài trăm mét (thường 300-400m trong data center) |
| Chi phí | Cáp rẻ hơn nhưng thiết bị phát (laser) đắt hơn | Cáp đắt hơn nhưng thiết bị phát rẻ hơn (LED chỉ dùng cho multi-mode tốc độ thấp ≤100 Mbps; multi-mode 1G/10G hiện đại dùng VCSEL — một loại laser giá rẻ) |
| Ứng dụng chính | Kết nối liên tòa nhà, backbone WAN/MAN, đường trục nhà mạng | Kết nối trong data center, giữa các tủ rack gần nhau |

> [!question] Vì sao multi-mode giới hạn khoảng cách ngắn hơn nhiều so với single-mode?
> Vì lõi lớn cho phép ánh sáng truyền theo nhiều "mode" (góc phản xạ khác nhau) cùng lúc — mỗi mode đi một quãng đường dài ngắn khác nhau nên đến đích lệch thời điểm nhau (hiện tượng modal dispersion), làm tín hiệu méo dần theo khoảng cách. Single-mode chỉ cho 1 tia truyền thẳng theo trục lõi nên gần như không bị hiện tượng này, đi được xa hơn rất nhiều.

> [!example] Case study — Đứt cáp quang biển Biển Đỏ (09/2025)
> Ngày 06/09/2025, một số tuyến cáp quang biển đi qua Biển Đỏ (SEA-ME-WE-4, IMEWE, FALCON GCX và theo ThousandEyes còn có EIG) bị đứt. Hậu quả: Microsoft Azure ở khu vực Trung Đông và châu Á bị tăng độ trễ đáng kể vì traffic phải định tuyến vòng qua các tuyến cáp khác dài hơn. Case study này — dù ở quy mô hạ tầng quốc gia — minh họa đúng nguyên lý vừa học ở quy mô nhỏ hơn trong doanh nghiệp: **phương tiện truyền dẫn vật lý là nền tảng của mọi thứ phía trên**; dù thiết kế định tuyến (Lớp 3) có tốt đến đâu, dữ liệu vẫn phải đi qua một phương tiện vật lý thật sự, và khi phương tiện đó đứt, mọi lớp phía trên đều bị ảnh hưởng dù bản thân chúng không có lỗi gì. Nguồn: [ThousandEyes](https://www.thousandeyes.com/blog/internet-report-red-sea-subsea-cable-cuts), [Network World](https://www.networkworld.com/article/4052813/red-sea-cable-cuts-trigger-latency-for-azure-cloud-services-across-asia-and-the-middle-east.html).

## 3. Phương tiện không dây

### 3.1 Bluetooth

Kết nối tầm ngắn (thường dưới 10-100m tùy class), băng tần 2.4 GHz, băng thông thấp. Dùng cho mạng cá nhân (PAN — Personal Area Network): chuột/bàn phím không dây, tai nghe, kết nối thiết bị IoT tầm gần. Không phải giải pháp cho hạ tầng LAN doanh nghiệp.

### 3.2 Radio (Wi-Fi — 802.11)

Sóng vô tuyến ở băng tần 2.4 GHz và 5 GHz (và 6 GHz với Wi-Fi 6E trở lên), là phương tiện không dây chính cho mạng LAN doanh nghiệp hiện nay. Đặc tính, chuẩn a/b/g/n/ac/ax, bảo mật WEP/WPA/WPA2/WPA3 được trình bày chi tiết ở Bài 24-25.

### 3.3 Viba (Microwave / Point-to-Point Wireless)

Kết nối điểm-điểm tầm xa (vài km đến vài chục km) bằng sóng vi ba tần số cao, cần 2 ăng-ten (dish) nhìn thấy nhau trực tiếp (line-of-sight) — không có vật cản như tòa nhà hoặc địa hình chắn giữa 2 đầu. Dùng phổ biến khi cần kết nối giữa 2 tòa nhà/chi nhánh mà không thể kéo cáp quang (địa hình khó, chi phí đào đường quá cao, hoặc cần triển khai nhanh) — một giải pháp thay thế cho thuê kênh WAN từ ISP trong một số trường hợp.

## 4. Các khái niệm nền tảng

### 4.1 Băng thông (Bandwidth) vs Thông lượng thực tế (Throughput)

**Băng thông** là khả năng tối đa lý thuyết của đường truyền (ví dụ cổng Gigabit = 1000 Mbps). **Throughput** là tốc độ thực tế đo được khi truyền dữ liệu thật — luôn thấp hơn bandwidth vì còn chịu ảnh hưởng của overhead giao thức, nhiễu, tắc nghẽn, chất lượng cáp. Ngộ nhận phổ biến nhất của người mới: nghĩ rằng cổng "1 Gbps" luôn cho tốc độ thực tế 1 Gbps — trên thực tế con số này chỉ là giới hạn trần, không phải cam kết.

### 4.2 Băng tần (Frequency band)

Dải tần số mà tín hiệu (đặc biệt là không dây) sử dụng để truyền — ví dụ Wi-Fi 2.4 GHz hay 5 GHz. Băng tần khác nhau có đặc tính lan truyền khác nhau: tần số càng thấp (2.4 GHz) càng xuyên vật cản tốt hơn nhưng băng thông thấp hơn và dễ bị nhiễu hơn (vì nhiều thiết bị gia dụng như lò vi sóng cũng dùng chung dải 2.4 GHz); tần số càng cao (5 GHz) băng thông lớn hơn nhưng xuyên tường kém hơn.

### 4.3 Nhiễu (Interference)

- **EMI (Electromagnetic Interference)**: nhiễu từ nguồn điện từ bên ngoài — động cơ, chấn lưu đèn huỳnh quang (ballast), đường dây điện cao thế chạy song song với cáp mạng.
- **RFI (Radio Frequency Interference)**: nhiễu từ sóng vô tuyến — thiết bị phát sóng gần đó.
- **Crosstalk**: tín hiệu từ cặp dây này "rò" sang cặp dây khác trong cùng một cáp, tăng mạnh khi đi cáp mật độ cao (phòng máy chủ, tủ rack) hoặc bó nhiều cáp sát nhau.

Hậu quả chung của các loại nhiễu: mất gói, tăng độ trễ, giảm tốc độ thực tế — dấu hiệu để phân biệt với lỗi cấu hình là: **triệu chứng xuất hiện/biến mất khi di chuyển hoặc thay thế đoạn cáp**, trong khi lỗi cấu hình thì không đổi dù đổi cáp.

### 4.4 Half-duplex vs Full-duplex

- **Half-duplex**: chỉ truyền hoặc nhận tại một thời điểm, không làm được cả hai cùng lúc (giống bộ đàm — nói xong phải buông nút mới nghe được). Đặc trưng của các mạng dùng Hub thời trước (mọi thiết bị chia sẻ chung 1 collision domain).
- **Full-duplex**: truyền và nhận đồng thời trên cùng cáp UTP (với 10/100BASE-T dùng các cặp dây riêng cho TX và RX; với 1000BASE-T/10GBASE-T dùng cả 4 cặp song hướng đồng thời nhờ kỹ thuật hybrid/triệt tiếng vọng — echo cancellation) — chuẩn mặc định của mọi kết nối Switch-to-PC hiện đại, loại bỏ hoàn toàn khái niệm đụng độ tín hiệu (collision) trên đoạn kết nối đó.

> [!question] Hai đầu kết nối lệch duplex (1 bên full, 1 bên half) thì sao?
> Đây là lỗi cấu hình kinh điển gọi là **duplex mismatch**: bên full-duplex gửi liên tục không chờ, bên half-duplex nghĩ đó là collision nên liên tục retransmit — kết quả là tốc độ thực tế giảm rất mạnh (có thể chỉ còn vài % băng thông danh nghĩa) kèm rất nhiều lỗi CRC/collision trên interface, dù đèn tín hiệu Lớp 1 vẫn sáng bình thường. Cách chẩn đoán: kiểm tra `show interface` trên switch, nếu thấy tốc độ/duplex hiển thị lệch với thiết bị đầu kia thì gần như chắc chắn là nguyên nhân — auto-negotiation hiện đại (802.3) đã giảm mạnh lỗi này nhưng vẫn xảy ra khi ép cứng (hard-code) tốc độ/duplex sai ở một đầu.

## 5. Bấm cáp UTP — Chuẩn T568A và T568B

Chuẩn TIA/EIA-568 quy định thứ tự 8 dây trong đầu RJ45:

| Pin | T568A | T568B |
|---|---|---|
| 1 | Trắng-Xanh lá | Trắng-Cam |
| 2 | Xanh lá | Cam |
| 3 | Trắng-Cam | Trắng-Xanh lá |
| 4 | Xanh dương | Xanh dương |
| 5 | Trắng-Xanh dương | Trắng-Xanh dương |
| 6 | Cam | Xanh lá |
| 7 | Trắng-Nâu | Trắng-Nâu |
| 8 | Nâu | Nâu |

- **Cáp thẳng (Straight-through)**: bấm cùng 1 chuẩn ở cả 2 đầu (T568B-T568B hoặc T568A-T568A) — dùng để nối 2 loại thiết bị khác nhau (PC ↔ Switch, Switch ↔ Router).
- **Cáp chéo (Crossover)**: một đầu T568A, một đầu T568B — theo lý thuyết dùng để nối 2 thiết bị cùng loại (PC ↔ PC, Switch ↔ Switch) vì phải hoán đổi cặp TX/RX cho đúng chiều.

> [!question] Vậy giờ còn cần bấm cáp chéo không?
> Hầu như không. Mọi switch/router/NIC sản xuất sau khoảng năm 2005 đều hỗ trợ **Auto-MDI/MDI-X** — tự động phát hiện và hoán đổi cặp TX/RX bên trong thiết bị. Trên thực tế hạ tầng hiện nay, một sợi cáp thẳng (T568B cả 2 đầu — chuẩn phổ biến nhất tại Việt Nam) dùng được cho mọi loại kết nối. Kiến thức về cáp chéo vẫn quan trọng để hiểu bản chất TX/RX và vẫn là câu hỏi phỏng vấn kinh điển.

## 6. Lab thực hành

**Môi trường**: kìm bấm mạng, đầu RJ45, cáp UTP, cable tester (nếu có điều kiện thực hành vật lý) — hoặc Cisco Packet Tracer để mô phỏng chọn đúng loại cáp giữa các thiết bị.

1. Bấm 1 sợi cáp thẳng theo chuẩn T568B ở cả 2 đầu, dùng cable tester kiểm tra đủ 8 dây thông mạch đúng thứ tự.
2. Trong Packet Tracer, thử nối PC-Switch bằng cáp thẳng (đúng) và bằng cáp chéo (thường bị báo lỗi hoặc cảnh báo loại cáp không phù hợp ở các phiên bản mô phỏng cũ chưa hỗ trợ Auto-MDI/MDI-X).
3. Thử nối Switch-Switch bằng cáp thẳng, quan sát đèn tín hiệu/trạng thái link — với thiết bị đời mới, kết nối vẫn lên bình thường nhờ Auto-MDI/MDI-X.

**Kết quả kỳ vọng**: phân biệt được bằng thực hành (không chỉ lý thuyết) khi nào cáp thẳng/chéo có ý nghĩa, và xác nhận cable tester giúp phát hiện lỗi bấm cáp (đứt dây, sai thứ tự) trước khi lắp đặt thật — một bước kiểm tra bắt buộc trong thi công hạ tầng thực tế, tránh phải tháo lắp lại sau khi đã đi cáp âm tường/trên trần.

## 7. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Cổng "1 Gbps" luôn cho tốc độ thực tế 1 Gbps | Bandwidth là giới hạn trần lý thuyết, không phải cam kết | Throughput thực tế luôn thấp hơn bandwidth do overhead, nhiễu, chất lượng cáp |
| Cat6a/Cat7 luôn tốt hơn nên cứ chọn loại cao nhất | Giá thành và độ cứng cáp tăng theo category, gây lãng phí nếu không cần chống nhiễu mạnh | Chọn category theo đúng khoảng cách/tốc độ/môi trường thực tế cần dùng, không phải cứ cao nhất là tối ưu |
| Multi-mode fiber đi xa như single-mode, chỉ là "cáp quang nào cũng vậy" | Nhầm lẫn giữa 2 loại có cơ chế truyền ánh sáng khác nhau hoàn toàn | Multi-mode giới hạn vài trăm mét (modal dispersion), single-mode đi được hàng chục-hàng trăm km |
| Vẫn cần bấm cáp chéo để nối Switch-Switch | Đúng theo lý thuyết cũ (trước 2005) nhưng không còn cần thiết với thiết bị hiện đại | Auto-MDI/MDI-X trên thiết bị đời mới tự hoán đổi TX/RX, cáp thẳng dùng được cho mọi kết nối |

## 8. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao cáp UTP dùng cấu trúc xoắn đôi (twisted pair) thay vì dây thẳng song song?**
→ Việc xoắn giúp 2 dây trong cùng cặp mang tín hiệu ngược pha, khi bị nhiễu bên ngoài tác động gần như đều lên cả 2 dây, nhiễu tự triệt tiêu (cancellation) ở đầu nhận — cơ chế chống nhiễu bị động, không cần lớp shield vật lý.

**2. Khi nào chọn STP thay vì UTP thường?**
→ Khi môi trường lắp đặt có nhiều nguồn nhiễu điện từ mạnh (nhà máy, phòng có động cơ/máy biến áp lớn, hoặc cáp buộc chạy song song gần đường điện) — nhưng phải đảm bảo tiếp đất đúng cách, nếu không lớp shield có thể phản tác dụng.

**3. Phân biệt single-mode và multi-mode fiber — khi nào dùng loại nào?**
→ Single-mode lõi nhỏ, 1 tia sáng, đi xa (hàng chục-hàng trăm km), dùng cho kết nối liên tòa nhà/backbone. Multi-mode lõi lớn, nhiều tia sáng gây modal dispersion nên giới hạn vài trăm mét, dùng trong data center giữa các rack gần nhau.

**4. Duplex mismatch là gì? Dấu hiệu nhận biết trên switch?**
→ Là lỗi cấu hình khi 2 đầu kết nối lệch chế độ duplex (1 full, 1 half). Dấu hiệu: tốc độ thực tế giảm mạnh, nhiều lỗi CRC/collision trên interface dù đèn tín hiệu vẫn sáng bình thường — kiểm tra bằng lệnh `show interface` để so sánh duplex/speed 2 đầu.

**5. Cáp thẳng và cáp chéo khác nhau ở đâu, ngày nay còn cần cáp chéo không?**
→ Cáp thẳng bấm cùng chuẩn T568A hoặc T568B ở cả 2 đầu; cáp chéo bấm khác chuẩn 2 đầu để hoán đổi cặp TX/RX. Ngày nay hầu như không cần cáp chéo nữa vì thiết bị hỗ trợ Auto-MDI/MDI-X tự hoán đổi.

## 9. Tài liệu tham khảo

- [Comprehensive Tutorial of Cat5e vs Cat6 vs Cat6A vs Cat7](https://www.qsfptek.com/qt-news/comprehensive-tutorial-cat5e-vs-cat6-vs-cat6a-vs-cat7.html)
- [T568A vs T568B: RJ45 Pinout — trueCABLE](https://www.truecable.com/blogs/cable-academy/t568a-vs-t568b)
- [Fiber Optic Cable Types: Single Mode vs Multimode — FS.com](https://www.fs.com/blog/fiber-optic-cable-types-single-mode-vs-multimode-fiber-cable-1310.html)
- [ThousandEyes — Red Sea Subsea Cable Cuts Internet Report](https://www.thousandeyes.com/blog/internet-report-red-sea-subsea-cable-cuts)
- [Network World — Red Sea cable cuts trigger latency for Azure](https://www.networkworld.com/article/4052813/red-sea-cable-cuts-trigger-latency-for-azure-cloud-services-across-asia-and-the-middle-east.html)

---
**Liên kết**: [[Bai-02_Mo-hinh-OSI-TCPIP|← Bài 2]] · [[Bai-04_Thiet-bi-ket-noi-mang|Bài 4 - Thiết bị kết nối mạng →]]
