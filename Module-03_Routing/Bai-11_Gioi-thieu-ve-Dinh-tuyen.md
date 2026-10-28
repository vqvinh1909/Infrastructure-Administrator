---
title: Bài 11 - Giới thiệu về Định tuyến
type: Concept
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — Routing table, Administrative Distance và phân loại giao thức định tuyến là kiến thức nền bắt buộc, luôn xuất hiện trong vòng phỏng vấn kỹ thuật đầu tiên"
tags: [networking, cisco, routing, routing-table, administrative-distance, sysadmin]
aliases: [Routing, Định tuyến, Routing Table, Administrative Distance]
related: ["[[Bai-05_Dia-chi-IPv4]]", "[[Bai-10_STP-va-EtherChannel]]"]
last_review: 2026-07-21
---

# Bài 11: Giới thiệu về Định tuyến

> [!important] Tại sao cần file này
> Toàn bộ Module 2 (Bài 7-10) giải quyết vấn đề trong phạm vi **1 broadcast domain** — dù có VLAN, trunk, hay EtherChannel, các thiết bị vẫn chỉ nói chuyện được với nhau nếu **cùng một mạng logic (subnet)**. [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8]] đã chủ động để lại câu hỏi mở: "2 PC khác VLAN không ping thông vì chưa có định tuyến liên VLAN". Bài này trả lời câu hỏi đó — Router (thiết bị Lớp 3 đã học sơ lược ở [[Bai-04_Thiet-bi-ket-noi-mang|Bài 4]]) là thiết bị duy nhất có khả năng chuyển tiếp gói tin **giữa các mạng logic khác nhau**, và routing table chính là "bản đồ" nó dùng để quyết định đường đi.

## 1. Vì sao cần định tuyến

Switch (Lớp 2) chuyển tiếp frame dựa trên MAC address, chỉ hoạt động được trong cùng 1 mạng logic (VLAN/subnet). Khi 2 thiết bị nằm ở 2 subnet khác nhau, switch **không có khả năng** đưa gói tin qua lại — bắt buộc phải có Router đứng giữa để **định tuyến (route)** gói tin dựa trên địa chỉ IP đích.

```
Row 1 - cung VLAN, khong can Router:
+--------------+    +------------+    +--------------+
|     PC-A     |    |   SWITCH   |    |     PC-B     |
|   VLAN 10    |----| (Layer 2)  |----|   VLAN 10    |
|  10.10.10.5  |    |            |    |  10.10.10.6  |
+--------------+    +------------+    +--------------+

Row 2 - khac VLAN, PHAI qua Router:
+--------------+  +------------+  +------------+  +------------+  +--------------+
|     PC-A     |  |   SWITCH   |  |   ROUTER   |  |   SWITCH   |  |     PC-B     |
|   VLAN 10    |--| (Layer 2)  |--| (Layer 3)  |--| (Layer 2)  |--|   VLAN 20    |
|  10.10.10.5  |  |            |  |            |  |            |  |  10.20.20.5  |
+--------------+  +------------+  +------------+  +------------+  +--------------+
```

> [!question] Vì sao switch "không có khả năng" định tuyến, dù về lý thuyết nó vẫn thấy được cả 2 gói tin?
> Vì switch đọc **Lớp 2 (MAC address)** để quyết định chuyển tiếp — nó không đọc địa chỉ IP đích và không có routing table. Ngay cả khi switch vật lý nhận được frame từ VLAN 10 lẫn VLAN 20, nó vẫn coi đây là 2 broadcast domain hoàn toàn tách biệt theo thiết kế (đã học ở [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8]], mục 1) — không có cơ chế "bắc cầu" giữa chúng nếu không phải là thiết bị Lớp 3.

## 2. Router hoạt động như thế nào — quy trình forwarding

Khi router nhận 1 gói tin (packet), nó thực hiện tuần tự:

1. Đọc **địa chỉ IP đích** trong IP header.
2. Tra trong **routing table**, tìm route khớp nhất với địa chỉ đích đó theo nguyên tắc **Longest Prefix Match** (route có subnet mask dài nhất/cụ thể nhất luôn được ưu tiên, bất kể route nào được học trước).
3. Nếu tìm thấy route khớp: chuyển gói tin ra đúng interface/next-hop tương ứng, giảm TTL đi 1.
4. Nếu **không** tìm thấy route khớp và không có default route (0.0.0.0/0): gói tin bị hủy, router gửi trả **ICMP Destination Unreachable** cho nguồn.

> [!question] Vì sao Longest Prefix Match quan trọng hơn cả Administrative Distance hay Metric?
> Vì đây là bước lọc **đầu tiên**. Ví dụ routing table có cả route `10.10.0.0/16` và `10.10.10.0/24` cùng dẫn đến đích `10.10.10.5` — router **luôn chọn `/24`** vì cụ thể hơn (prefix dài hơn), bất kể route `/16` có Administrative Distance tốt hơn thế nào. AD (mục 4) và Metric chỉ được dùng để so sánh giữa các route **có cùng độ dài prefix**.

## 3. Routing table — cách đọc

Lệnh `show ip route` hiển thị toàn bộ routing table. Ví dụ output thực tế:

```
Codes: L - local, C - connected, S - static, O - OSPF, D - EIGRP

Gateway of last resort is 10.10.10.2 to network 0.0.0.0

C     10.10.10.0/24 is directly connected, GigabitEthernet0/0
L     10.10.10.1/32 is directly connected, GigabitEthernet0/0
S     20.20.20.0/24 [1/0] via 10.10.10.2
O     30.30.30.0/24 [110/2] via 10.10.10.2, 00:05:12, GigabitEthernet0/0
S*    0.0.0.0/0 [1/0] via 10.10.10.2
```

| Ký hiệu | Ý nghĩa |
|---|---|
| `C` | Connected — mạng gắn trực tiếp vào 1 interface của router đang bật (up/up) |
| `L` | Local — địa chỉ IP /32 của chính interface router đó (tự động sinh từ IOS 15 trở lên) |
| `S` | Static — route cấu hình tay ([[Bai-13_Static-Route|Bài 13]]) |
| `S*` | Static route, đồng thời là **default route** (Gateway of last resort) |
| `O` | OSPF — route học được qua giao thức OSPF ([[Bai-14_OSPF|Bài 14]]) |
| `D` | EIGRP — route học được qua giao thức EIGRP ([[Bai-15_EIGRP|Bài 15]]) |

Cấu trúc `[AD/Metric]` (ví dụ `[110/2]`) sẽ được giải thích ở mục 4.

## 4. Administrative Distance (AD) và Metric

Khi routing table nhận được **nhiều route cùng đích, cùng độ dài prefix** nhưng từ **nguồn khác nhau** (ví dụ vừa có static route vừa có route học qua OSPF cho cùng 1 mạng), router phải chọn 1 route duy nhất để đưa vào bảng — quyết định dựa trên 2 tiêu chí theo đúng thứ tự ưu tiên:

**Bước 1 — Administrative Distance (AD)**: độ "tin cậy" của nguồn route, **AD càng nhỏ càng được ưu tiên**. Đây là con số cấu hình sẵn trên router (có thể chỉnh tay), không phải một phép đo thực tế.

| Nguồn route | AD mặc định (Cisco) |
|---|---|
| Connected | 0 |
| Static | 1 |
| EIGRP (summary route) | 5 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| iBGP | 200 |
| Unknown/không tin cậy | 255 (không bao giờ được đưa vào routing table) |

**Bước 2 — Metric**: chỉ so sánh khi **2 route có cùng AD** (ví dụ 2 route OSPF cùng đích qua 2 đường khác nhau) — metric càng nhỏ càng được ưu tiên. Mỗi giao thức tự định nghĩa cách tính metric riêng (OSPF dùng cost dựa trên băng thông, EIGRP dùng công thức kết hợp băng thông + delay — chi tiết ở [[Bai-14_OSPF|Bài 14]] và [[Bai-15_EIGRP|Bài 15]]).

> [!question] Nếu static route và OSPF cùng quảng bá 1 mạng đích giống hệt nhau (cùng prefix), route nào thắng?
> Static route thắng, vì AD = 1 nhỏ hơn AD = 110 của OSPF — bất kể route OSPF có "tốt" hơn về mặt băng thông thực tế. Đây là lý do khi cấu hình static route sai (trỏ nhầm next-hop) trong lúc router vẫn đang chạy giao thức động, router vẫn **ưu tiên dùng static route sai đó** — một lỗi thực chiến rất dễ gặp và khó nhận ra vì không có cảnh báo nào cả.

## 5. Phân loại nguồn route: Directly Connected, Static, Dynamic

- **Directly Connected**: mạng gắn trực tiếp vào interface đang up/up của router — tự động xuất hiện trong routing table (ký hiệu `C`/`L`), không cần cấu hình thêm gì ngoài đặt IP cho interface.
- **Static Route**: quản trị viên tự tay khai báo đường đi cụ thể — router không tự học, không tự thích nghi khi mạng thay đổi. Phù hợp mạng nhỏ, ổn định, hoặc dùng làm route dự phòng (chi tiết [[Bai-13_Static-Route|Bài 13]]).
- **Dynamic Route**: router tự động trao đổi thông tin định tuyến với nhau qua 1 giao thức chung, tự thích nghi khi topology thay đổi (đứt cáp, thêm mạng mới) mà không cần cấu hình lại tay. Đánh đổi: tốn tài nguyên CPU/băng thông cho việc trao đổi, và phức tạp hơn để thiết kế/troubleshoot đúng.

**Ưu và nhược điểm — Static Route so với Dynamic Route:**

| Tiêu chí | Static Route | Dynamic Route |
|---|---|---|
| Độ phức tạp cấu hình | Đơn giản, dễ hiểu, dễ kiểm soát | Phức tạp hơn, cần hiểu cơ chế giao thức để thiết kế đúng |
| Tài nguyên tiêu tốn | Không tốn CPU/băng thông cho trao đổi định tuyến | Tốn thêm CPU/băng thông cho Hello, cập nhật, tính toán lại |
| Khả năng thích nghi | Không tự thích nghi — mạng thay đổi thì phải sửa tay | Tự động phát hiện và thích nghi khi topology thay đổi |
| Bảo mật/khả năng kiểm soát | An toàn hơn vì không quảng bá thông tin định tuyến ra ngoài | Có thể bị khai thác nếu không cấu hình xác thực (authentication) đúng cách |
| Khả năng mở rộng | Không phù hợp mạng lớn, nhiều route (tốn công sửa tay) | Phù hợp mạng lớn, nhiều thay đổi topology |
| Trường hợp dùng phổ biến | Mạng nhỏ, ổn định, mạng stub (chỉ có 1 đường ra), hoặc làm route dự phòng | Mạng doanh nghiệp vừa/lớn, có nhiều đường dự phòng cần tự động chuyển đổi |

## 6. Phân loại giao thức định tuyến động

**Theo phạm vi hoạt động:**

| Loại | Ý nghĩa | Ví dụ |
|---|---|---|
| **IGP** (Interior Gateway Protocol) | Định tuyến **bên trong** 1 tổ chức/AS (Autonomous System) duy nhất | RIP, OSPF, EIGRP, IS-IS |
| **EGP** (Exterior Gateway Protocol) | Định tuyến **giữa** các tổ chức/AS khác nhau — dùng cho Internet backbone | BGP |

**Theo cơ chế hoạt động:**

| Loại | Cách hoạt động | Ví dụ |
|---|---|---|
| **Distance Vector** | Mỗi router chỉ biết "hướng đi và khoảng cách" tới đích qua thông tin hàng xóm gửi lại, không biết toàn bộ topology | RIP |
| **Link State** | Mỗi router xây dựng **bản đồ đầy đủ topology** toàn vùng bằng cách trao đổi thông tin về từng liên kết (link), sau đó tự chạy thuật toán Dijkstra để tính đường ngắn nhất | OSPF, IS-IS |
| **Hybrid (Advanced Distance Vector)** | Kết hợp đặc điểm 2 loại trên — dùng thuật toán DUAL để hội tụ nhanh và tránh loop mà không cần xây bản đồ đầy đủ như Link State | EIGRP |

Module 3 (Bài 13-15) sẽ đi sâu vào Static Route, OSPF (Link State) và EIGRP (Hybrid) — 2 giao thức động phổ biến nhất trong hạ tầng doanh nghiệp thực tế hiện nay.

## 7. Các loại thiết bị hỗ trợ định tuyến

Router không phải là thiết bị duy nhất có khả năng định tuyến trong hạ tầng doanh nghiệp thực tế:

| Thiết bị | Vai trò định tuyến |
|---|---|
| **Router** (chuyên dụng) | Thiết bị Lớp 3 thuần túy, tối ưu cho định tuyến WAN/Internet, thường đặt ở vị trí biên (edge) kết nối ra ngoài |
| **Layer 3 Switch (Multilayer Switch)** | Vừa chuyển mạch Lớp 2 vừa định tuyến Lớp 3 ngay trên phần cứng (ASIC) — phổ biến ở lớp Distribution/Core trong mạng doanh nghiệp (chi tiết SVI ở [[Bai-12_Inter-VLAN-Routing|Bài 12]]) |
| **Firewall thế hệ mới (NGFW)** | Nhiều firewall hiện đại (ví dụ Palo Alto, Fortinet, Cisco Firepower) tích hợp sẵn khả năng định tuyến động (OSPF/BGP) ngoài chức năng lọc traffic, thường đặt ở vị trí biên mạng |
| **Router phần mềm (Software Router)** | Máy chủ Linux/Windows Server cấu hình định tuyến bằng phần mềm (ví dụ Linux với `ip route`, FRRouting) — ít phổ biến ở doanh nghiệp lớn nhưng vẫn gặp trong môi trường lab, cloud, hoặc mạng nhỏ chi phí thấp |

> [!question] Vì sao doanh nghiệp lớn thường ưu tiên Layer 3 Switch cho định tuyến nội bộ, còn Router chuyên dụng chỉ giữ vai trò biên?
> Vì Layer 3 Switch định tuyến bằng phần cứng (ASIC) ở tốc độ line-rate, phù hợp khối lượng traffic nội bộ lớn giữa các VLAN (đã phân tích ở [[Bai-12_Inter-VLAN-Routing|Bài 12]] mục 3). Router chuyên dụng lại mạnh hơn về các tính năng đặc thù kết nối WAN (mã hóa, nén, giao thức WAN, xử lý gói tin phức tạp hơn bằng phần mềm) — nên thường chỉ đảm nhận vai trò router biên kết nối ra ISP/chi nhánh, trong khi định tuyến nội bộ giữa các VLAN được giao cho Layer 3 Switch xử lý.

## 8. Cấu hình Router cơ bản

```
Router> enable
Router# configure terminal
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ip address 10.10.10.1 255.255.255.0
Router(config-if)# no shutdown
Router(config-if)# exit
```

> [!question] Vì sao interface router mặc định ở trạng thái `shutdown`, khác với switch?
> Đây là khác biệt thiết kế giữa 2 dòng thiết bị: interface switch (Layer 2) mặc định **up** vì switch chủ yếu chuyển tiếp ngay khi cắm dây; interface router mặc định **shutdown** vì cấu hình IP sai trên 1 router có thể gây ảnh hưởng định tuyến cho toàn mạng — buộc quản trị viên phải chủ động `no shutdown` sau khi đã chắc chắn cấu hình đúng, như một lớp an toàn bổ sung.

**Sao lưu và phục hồi cấu hình Router:**

```
Router# copy running-config startup-config
Router# copy running-config tftp
Router# copy tftp running-config
```

- `copy running-config startup-config`: lưu cấu hình đang chạy (RAM) vào bộ nhớ khởi động (NVRAM) — bắt buộc phải chạy sau khi cấu hình xong, nếu không toàn bộ thay đổi sẽ **mất khi router khởi động lại**.
- `copy running-config tftp`: sao lưu cấu hình ra 1 máy chủ TFTP bên ngoài — thực hành chuẩn để có bản backup off-device, phòng trường hợp router hỏng hoàn toàn.
- `copy tftp running-config`: khôi phục cấu hình từ bản backup TFTP về router — dùng khi cần phục hồi nhanh sau sự cố hoặc khi thay thế thiết bị mới.

> [!question] Vì sao "backup off-device" (ra TFTP/server ngoài) quan trọng hơn chỉ lưu `startup-config` trên chính router?
> Vì `startup-config` vẫn nằm trên **chính thiết bị đó** — nếu router hỏng phần cứng hoàn toàn (cháy nguồn, hỏng NVRAM), cấu hình đã lưu cũng mất theo cùng thiết bị. Backup ra TFTP/server ngoài đảm bảo có 1 bản sao độc lập, có thể phục hồi ngay cả khi phải thay thế bằng 1 router hoàn toàn mới — đây là thực hành bắt buộc trong quy trình Disaster Recovery của doanh nghiệp (sẽ học sâu hơn ở Module Monitoring & Backup).

## 9. Case study — AS 7007 (1997): hậu quả khi thông tin định tuyến sai lan rộng

> [!example] Case study — Sự cố AS7007, một trong những sự cố định tuyến lớn đầu tiên của Internet
> Ngày 25/4/1997, một lỗi phần mềm trên router của AS7007 (MAI Network Services) khiến router này **quảng bá nhầm gần như toàn bộ bảng định tuyến Internet** (BGP — giao thức EGP đã nhắc ở mục 6) như thể chính AS7007 là nguồn gốc của các dải IP đó. Vì các route bị quảng bá nhầm này có prefix **cụ thể hơn** (dài hơn) so với route hợp lệ đang tồn tại, theo đúng nguyên tắc Longest Prefix Match (mục 2), phần lớn Internet toàn cầu bắt đầu định tuyến traffic **về đúng AS7007** — nơi thiết bị của họ quá tải và làm rớt gần như toàn bộ traffic đó, tạo thành 1 "hố đen định tuyến" (routing black hole). Sự cố này kéo dài nhiều giờ, ngay cả sau khi router lỗi bị ngắt khỏi mạng vì các route sai vẫn tiếp tục lan truyền trong một thời gian.
>
> **Bài học thực chiến**: dù đây là sự cố ở tầng EGP (BGP, ngoài phạm vi Module 3 vốn tập trung IGP), nguyên lý gốc rễ hoàn toàn áp dụng được cho cả OSPF/EIGRP nội bộ — **Longest Prefix Match luôn được ưu tiên tuyệt đối**, và một router quảng bá sai thông tin định tuyến (dù vô tình do lỗi phần mềm hay do cấu hình tay sai) có thể gây ảnh hưởng vượt xa phạm vi router đó. Đây chính là lý do các giao thức định tuyến hiện đại đều có cơ chế xác thực (authentication) và lọc route (route filtering/summarization) — sẽ gặp lại khái niệm này khi học OSPF/EIGRP. Nguồn: [Wikipedia — AS 7007 incident](https://en.wikipedia.org/wiki/AS_7007_incident), [Kentik — A Brief History of the Internet's Biggest BGP Incidents](https://www.kentik.com/blog/a-brief-history-of-the-internets-biggest-bgp-incidents/).

## 10. Verify & Troubleshoot

| Lệnh | Công dụng |
|---|---|
| `show ip route` | Xem toàn bộ routing table hiện tại |
| `show ip interface brief` | Xem nhanh trạng thái IP/up-down của từng interface |
| `show ip protocols` | Xem các giao thức định tuyến động đang chạy trên router |
| `traceroute <IP đích>` | Xác định gói tin đi qua những router nào trước khi tới đích — hữu ích khi nghi ngờ định tuyến sai đường |

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| 2 PC khác subnet không ping thông dù router đã lên đủ | Thiếu route tới subnet đích trong routing table, hoặc interface router chưa `no shutdown` | `show ip route` kiểm tra route đích có tồn tại; `show ip interface brief` kiểm tra interface đã up/up |
| Traffic đi "vòng xa" hơn dự kiến dù có đường ngắn hơn | Route ngắn hơn có AD cao hơn route dài hơn đang tồn tại (mục 4), hoặc route ngắn bị thiếu do lỗi cấu hình | Đối chiếu `show ip route`, kiểm tra Longest Prefix Match và AD của các route liên quan |
| Router không nhận traffic dù đã cấu hình đúng IP | Quên `no shutdown` — mặc định interface router luôn tắt (mục 8) | `show ip interface brief`, tìm interface ở trạng thái `administratively down` |
| Route static và route động cùng đích nhưng chỉ 1 route xuất hiện trong bảng | Đúng hành vi thiết kế — router chỉ giữ route có AD thấp nhất cho mỗi đích (mục 4), không phải lỗi | Không cần xử lý; nếu muốn ưu tiên khác, chỉnh AD tay hoặc gỡ route không mong muốn |

## 11. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 2 router + 2 PC ở 2 subnet khác nhau.

1. Cấu hình IP cho từng router theo mục 8, xác nhận PC cùng subnet với mỗi router ping thông router của mình.
2. Chạy `show ip route` trên cả 2 router, quan sát các route `C`/`L` tự động xuất hiện.
3. 2 PC ở 2 subnet khác nhau (nối qua 2 router riêng) sẽ **chưa** ping thông nhau — đây là điểm khởi đầu để học Static Route ở [[Bai-13_Static-Route|Bài 13]].
4. Thử tắt interface 1 router bằng `shutdown`, quan sát route `C`/`L` tương ứng biến mất khỏi `show ip route` — xác nhận mối liên hệ trực tiếp giữa trạng thái interface và routing table.

**Kết quả kỳ vọng**: hiểu được vì sao chỉ đặt IP thôi chưa đủ để 2 mạng khác nhau nói chuyện được — cần thêm 1 nguồn route (static hoặc dynamic) trỏ đường đi cụ thể, nội dung sẽ học ở các bài tiếp theo.

## 12. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Đặt IP đúng subnet cho router là đủ để 2 mạng khác nhau thông nhau | Bỏ qua việc router chỉ tự biết mạng **trực tiếp kết nối**, không tự biết mạng ở xa | Cần thêm route (static hoặc dynamic) để router biết đường tới mạng không trực tiếp kết nối |
| AD thấp nghĩa là "đường đi nhanh hơn" | Nhầm AD (độ tin cậy nguồn) với Metric (chất lượng đường đi) | AD chỉ quyết định "tin nguồn nào hơn"; Metric mới thực sự đo chất lượng đường đi, chỉ so sánh khi AD bằng nhau |
| Route có prefix ngắn hơn nhưng AD thấp hơn sẽ luôn thắng route prefix dài hơn AD cao hơn | Bỏ qua thứ tự ưu tiên Longest Prefix Match luôn được xét trước AD | Router luôn ưu tiên prefix dài hơn (cụ thể hơn) trước, chỉ dùng AD/Metric để so sánh giữa các route **cùng độ dài prefix** |
| Interface router mặc định đã bật sẵn giống switch | Nhầm hành vi mặc định giữa 2 dòng thiết bị | Interface router mặc định `shutdown`, luôn phải `no shutdown` tường minh sau khi cấu hình IP |

## 13. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao switch không thể định tuyến được giữa 2 VLAN dù cùng cắm chung 1 thiết bị vật lý?**
→ Vì switch (thuần Lớp 2) chuyển tiếp dựa trên MAC address và coi mỗi VLAN là 1 broadcast domain tách biệt hoàn toàn theo thiết kế — nó không đọc địa chỉ IP đích và không có routing table để "bắc cầu" giữa các subnet khác nhau.

**2. Longest Prefix Match là gì, và nó liên quan gì đến Administrative Distance?**
→ Longest Prefix Match là nguyên tắc router luôn ưu tiên route có subnet mask dài nhất (cụ thể nhất) khớp với địa chỉ đích, được xét **trước** AD. AD chỉ được dùng để so sánh khi có nhiều route cùng đích và cùng độ dài prefix nhưng khác nguồn.

**3. Administrative Distance và Metric khác nhau ở điểm nào?**
→ AD là con số cấu hình sẵn thể hiện độ tin cậy của nguồn route (static, OSPF, EIGRP...), càng nhỏ càng ưu tiên, không đo chất lượng đường đi thực tế. Metric là giá trị mỗi giao thức tự tính để so sánh chất lượng giữa các đường đi cùng nguồn, chỉ so sánh khi AD bằng nhau.

**4. IGP và EGP khác nhau ở điểm nào? Cho ví dụ.**
→ IGP định tuyến bên trong 1 AS (tổ chức) duy nhất, ví dụ OSPF, EIGRP. EGP định tuyến giữa các AS khác nhau, dùng cho Internet backbone, ví dụ BGP.

**5. Vì sao sự cố AS7007 lại nghiêm trọng đến mức ảnh hưởng gần như toàn bộ Internet?**
→ Vì router lỗi quảng bá các route với prefix cụ thể hơn (dài hơn) route hợp lệ đang tồn tại, khiến theo đúng nguyên tắc Longest Prefix Match, phần lớn Internet định tuyến nhầm traffic về đúng thiết bị đó — gây quá tải và làm rớt traffic trên diện rộng, minh chứng cho việc thông tin định tuyến sai có thể lan xa hơn phạm vi thiết bị gây lỗi.

## 14. Tài liệu tham khảo

- [Cisco — Describe Administrative Distance](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/15986-admin-distance.html)
- [Wikipedia — Administrative distance](https://en.wikipedia.org/wiki/Administrative_distance)
- [Wikipedia — AS 7007 incident](https://en.wikipedia.org/wiki/AS_7007_incident)
- [Kentik — A Brief History of the Internet's Biggest BGP Incidents](https://www.kentik.com/blog/a-brief-history-of-the-internets-biggest-bgp-incidents/)

---
**Liên kết**: [[Bai-10_STP-va-EtherChannel|← Bài 10]] · [[Bai-12_Inter-VLAN-Routing|Bài 12 - Inter-VLAN Routing →]]
