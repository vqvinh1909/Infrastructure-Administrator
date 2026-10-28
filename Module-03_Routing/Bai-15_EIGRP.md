---
title: Bài 15 - EIGRP
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — EIGRP là giao thức Hybrid độc quyền Cisco phổ biến trong mạng doanh nghiệp toàn thiết bị Cisco, thường được hỏi so sánh trực tiếp với OSPF"
tags: [networking, cisco, routing, eigrp, dual, hybrid, sysadmin]
aliases: [EIGRP, DUAL, Feasible Successor, Successor]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-14_OSPF]]"]
last_review: 2026-07-21
---

# Bài 15: EIGRP (Enhanced Interior Gateway Routing Protocol)

> [!important] Tại sao cần file này
> [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] (mục 6) đã phân loại EIGRP là giao thức **Hybrid (Advanced Distance Vector)** — không xây dựng bản đồ topology đầy đủ như OSPF ([[Bai-14_OSPF|Bài 14]]), nhưng vẫn hội tụ nhanh và tránh loop nhờ thuật toán **DUAL**. Đây là giao thức định tuyến động phổ biến thứ 2 sau OSPF trong hạ tầng doanh nghiệp, đặc biệt phổ biến ở các tổ chức dùng toàn bộ thiết bị Cisco vì lịch sử độc quyền hãng của nó (đã mở một phần qua RFC 7868 nhưng vẫn chủ yếu gắn liền hệ sinh thái Cisco).

## 1. Thuật toán DUAL — Successor và Feasible Successor

EIGRP dùng **DUAL (Diffusing Update Algorithm)** để tính và lưu sẵn đường dự phòng **không cần tính toán lại từ đầu** khi đường chính hỏng — đây chính là lý do EIGRP hội tụ nhanh dù không xây bản đồ đầy đủ như Link State. 2 khái niệm cốt lõi:

- **Feasible Distance (FD)**: tổng metric tốt nhất từ router hiện tại tới đích, tính qua đường đang dùng.
- **Reported Distance (RD)** — còn gọi là **Advertised Distance (AD)** trong tài liệu Cisco cũ hơn: metric mà 1 router hàng xóm quảng bá để tới cùng đích đó (từ góc nhìn của chính hàng xóm đó). Đây là 2 tên gọi cho **cùng 1 khái niệm** — tài liệu/chứng chỉ hiện đại thường dùng "Reported Distance (RD)" để tránh nhầm với "Administrative Distance (AD)" của [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] (mục 4) vốn viết tắt trùng là "AD" nhưng là khái niệm hoàn toàn khác.

```
+----------+                                   +----------+
|    R1    |==== qua R2, FD=30 (Successor) ====| Mang X   |
|          |-- qua R3, RD=25 (Feasible Succ) --|          |
+----------+                                   +----------+
```

- **Successor**: đường đi tốt nhất hiện tại (FD nhỏ nhất) — được đưa vào routing table, dùng để forward traffic thật.
- **Feasible Successor (FS)**: đường dự phòng đã được **tính sẵn trước** (pre-computed) — không cần đợi tới lúc Successor hỏng mới tính. Điều kiện để 1 hàng xóm được công nhận là Feasible Successor gọi là **Feasibility Condition (FC)**:

> [!question] Feasibility Condition là gì, và vì sao nó đảm bảo đường dự phòng không bị loop?
> Feasibility Condition yêu cầu: **RD của hàng xóm đó phải nhỏ hơn FD hiện tại** (RD < FD). Về bản chất, điều kiện này đảm bảo hàng xóm được chọn làm Feasible Successor **không đi qua chính router hiện tại** để tới đích — vì nếu hàng xóm đó cũng phải đi vòng qua router hiện tại, RD của nó chắc chắn sẽ lớn hơn hoặc bằng FD (do phải cộng thêm chi phí đi qua router hiện tại). Nhờ điều kiện toán học đơn giản này, Feasible Successor **luôn đảm bảo loop-free** mà không cần router phải xây dựng bản đồ topology đầy đủ như OSPF.

Khi Successor hỏng và **có sẵn** Feasible Successor thỏa điều kiện trên, router chuyển ngay sang dùng FS mà **không cần gửi Query** hỏi các router khác — đây là trường hợp hội tụ nhanh nhất của EIGRP. Nếu **không có** Feasible Successor nào khả dụng, router phải chuyển sang trạng thái **Active**, gửi Query hỏi toàn bộ hàng xóm liên quan để tìm đường thay thế — quá trình này chậm hơn nhiều và là nguồn gốc của vấn đề "Stuck In Active" (mục 9).

## 2. Metric của EIGRP

EIGRP dùng **công thức metric tổng hợp (composite metric)** dựa trên 4 yếu tố: Bandwidth, Delay, Load, Reliability (hệ số K1..K5; MTU chỉ được truyền kèm trong thông tin route, không tham gia công thức metric) — nhưng **mặc định chỉ Bandwidth và Delay được bật** (thông qua hệ số K1=1, K3=1; K2=K4=K5=0).

Công thức rút gọn với K-values mặc định:

```
Metric = 256 * [ (10^7 / Bandwidth nhỏ nhất trên đường đi, tính bằng kbps) + Tổng Delay (đơn vị 10 microsecond) ]
```

> [!question] Vì sao EIGRP chỉ lấy Bandwidth **nhỏ nhất** trên toàn đường đi, chứ không cộng dồn như Delay?
> Vì Bandwidth đại diện cho "cổ chai" (bottleneck) của cả tuyến đường — dù các đoạn khác có băng thông cao đến đâu, tốc độ thực tế của cả tuyến vẫn bị giới hạn bởi đoạn chậm nhất. Delay ngược lại mang tính **cộng dồn hợp lý** — thời gian trễ thực tế của cả tuyến đúng bằng tổng độ trễ từng chặng cộng lại, nên EIGRP cộng dồn giá trị này qua từng hop.

Nhắc lại từ [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] (mục 4): Administrative Distance mặc định của EIGRP là **90** cho route nội bộ (internal), **170** cho route ngoại bộ (external, học từ redistribution), và **5** cho route summary — trong đó internal (90) và summary (5) thấp hơn OSPF (110), còn external (170) thì cao hơn; nghĩa là nếu cùng khai báo 1 đích giống hệt nhau, route EIGRP internal sẽ được ưu tiên hơn route OSPF.

## 3. Điều kiện thiết lập Neighbor (bao gồm AS Number)

EIGRP chỉ hình thành neighbor khi **toàn bộ** các điều kiện sau khớp nhau giữa 2 router:

| Điều kiện | Chi tiết |
|---|---|
| **AS Number** | Bắt buộc khớp chính xác — khác OSPF (Process ID chỉ có ý nghĩa cục bộ, không cần khớp, đã học ở [[Bai-14_OSPF|Bài 14]] mục 8) |
| **K-values** | Hệ số K1-K5 dùng trong công thức metric (mục 2) phải giống nhau 2 đầu; mặc định đều giống nhau nếu không ai chỉnh tay |
| **Subnet chính (primary)** | 2 interface phải cùng subnet chính; subnet phụ (secondary) không được xét |
| **Authentication** | Nếu 1 đầu bật xác thực, đầu kia phải cấu hình khớp (cùng phương thức, cùng khóa) |

```
Router(config)# router eigrp 100
```

Số `100` ở đây là AS Number — router khác cấu hình `router eigrp 200` sẽ **không bao giờ** hình thành neighbor với router này, dù mọi thông số khác (IP, mask, interface) đều đúng.

> [!question] Điều gì xảy ra nếu 2 router láng giềng cấu hình sai lệch AS Number, và có cảnh báo gì không?
> Router sẽ **không hình thành neighbor**, nhưng quan trọng hơn: **không có bất kỳ syslog cảnh báo nào** báo hiệu nguyên nhân là do sai AS Number — đây là lỗi cấu hình rất dễ gặp và khó chẩn đoán với người mới, vì mọi thứ nhìn qua có vẻ đúng (IP đúng subnet, interface up) nhưng EIGRP vẫn im lặng không hình thành quan hệ. Đây là lý do khi troubleshooting EIGRP không lên neighbor, việc đầu tiên cần đối chiếu chính là AS Number ở cả 2 đầu, tương tự cách đối chiếu Area ID/timer của OSPF ([[Bai-14_OSPF|Bài 14]] mục 4).

## 4. Cấu hình EIGRP cơ bản cho IPv4 (Classic Mode)

```
Router(config)# router eigrp 100
Router(config-router)# network 10.10.10.0 0.0.0.255
Router(config-router)# network 10.10.20.0 0.0.0.255
Router(config-router)# no auto-summary
```

`no auto-summary` tắt hành vi tự động tóm tắt route theo lớp mạng cổ điển (classful) — mặc định EIGRP cổ điển có xu hướng gộp route theo classful boundary, thường gây mất thông tin subnet chi tiết trong mạng dùng VLSM; nên gần như luôn cần tắt trong thiết kế mạng hiện đại. Lưu ý phụ thuộc phiên bản: auto-summary đã TẮT mặc định từ IOS 15.0(1)M (và 12.2(33)SRE); chỉ IOS cũ hơn mới mặc định bật, nên vẫn nên khai báo `no auto-summary` tường minh để đồng nhất trên mọi thiết bị.

## 5. Cấu hình EIGRP cho IPv6

EIGRP for IPv6 dùng cú pháp riêng, **không có lệnh `network`** — thay vào đó kích hoạt trực tiếp trên từng interface, tương tự cách OSPFv3 hoạt động ([[Bai-14_OSPF|Bài 14]] mục 9):

```
Router(config)# ipv6 unicast-routing
Router(config)# ipv6 router eigrp 100
Router(config-rtr)# eigrp router-id 1.1.1.1
Router(config-rtr)# no shutdown
Router(config-rtr)# exit
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ipv6 eigrp 100
```

> [!question] Vì sao EIGRP for IPv6 cần lệnh `no shutdown` ngay trong chế độ `router eigrp`, điều không tồn tại ở EIGRP IPv4?
> Đây là khác biệt thiết kế: tiến trình EIGRP for IPv6 mặc định khởi tạo ở trạng thái **shutdown** để tránh vô tình chạy định tuyến khi cấu hình chưa hoàn chỉnh (đặc biệt vì EIGRP IPv6 không cần địa chỉ IPv6 toàn cục để hoạt động, nên rất dễ vô tình "bật nhầm" nếu không có lớp an toàn này) — quản trị viên phải chủ động `no shutdown` sau khi đã chắc chắn cấu hình xong, giống tinh thần "an toàn trước" đã thấy ở việc interface router mặc định `shutdown` tại [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 8.

## 6. Variance — Cân bằng tải Unequal-Cost

Mặc định EIGRP chỉ đưa vào routing table các route có metric **bằng nhau tuyệt đối** (equal-cost, giống OSPF). Lệnh `variance` cho phép EIGRP đưa thêm cả các Feasible Successor có metric **cao hơn** Successor (trong giới hạn hệ số nhân) vào routing table để cùng chia tải:

```
Router(config)# router eigrp 100
Router(config-router)# variance 3
```

`variance 3` nghĩa là: mọi Feasible Successor có metric **nhỏ hơn 3 lần** Feasible Distance của Successor sẽ được đưa vào routing table để cùng chia sẻ traffic — ví dụ nếu Successor có FD = 30, mọi Feasible Successor có metric dưới 90 (30 × 3) đều được dùng đồng thời. `variance 1` (mặc định) chỉ cho phép cân bằng tải equal-cost.

> [!question] Traffic được chia theo tỷ lệ nào khi dùng `variance` với 2 đường có metric chênh lệch nhau?
> EIGRP chia tải theo tỷ lệ **nghịch đảo** với chênh lệch metric — ví dụ nếu 1 đường có metric gấp 5 lần đường còn lại, traffic sẽ được chia theo tỷ lệ 5:1 (đường metric thấp hơn nhận nhiều traffic hơn hẳn), không chia đều 50/50 giữa 2 đường bất kể chênh lệch chất lượng. Điều quan trọng: route đó vẫn **phải thỏa Feasibility Condition** (mục 1) trước khi được `variance` xem xét — `variance` không phá vỡ nguyên tắc loop-free của DUAL.

## 7. Default Route trong EIGRP

EIGRP không có lệnh tương đương `default-information originate` của OSPF — có nhiều cách để quảng bá default route (`0.0.0.0/0`, đã học ở [[Bai-13_Static-Route|Bài 13]] mục 4) vào EIGRP:

```
Router(config)# ip route 0.0.0.0 0.0.0.0 <next-hop ra ngoài>
Router(config)# router eigrp 100
Router(config-router)# redistribute static metric 10000 100 255 1 1500
```

Cách này **redistribute** (tái phân phối) static default route đã có sẵn vào EIGRP — router khác trong vùng sẽ học được route `0.0.0.0/0` như 1 route EIGRP external (AD = 170, mục 2).

> **Lưu ý cập nhật:** Ngoài `redistribute static`, còn các cách khác: `network 0.0.0.0` (khi router có sẵn default route trong bảng định tuyến), `ip summary-address eigrp <AS> 0.0.0.0 0.0.0.0` trên interface (thường được khuyến nghị hơn), và `ip default-network`. Với default route được redistribute, `show ip route` hiển thị dạng `D*EX` (có dấu `*` đánh dấu candidate default).

> [!question] Vì sao `redistribute static` cần khai báo thêm 5 con số metric (`10000 100 255 1 1500`) ngay sau nó?
> Vì static route (nguồn ngoài EIGRP) không tự có sẵn 5 thành phần metric mà EIGRP cần (Bandwidth, Delay, Reliability, Load, MTU — lưu ý MTU chỉ được truyền kèm trong thông tin route, không tham gia công thức metric, xem mục 2) — khi redistribute 1 route từ nguồn khác vào EIGRP, phải khai báo tay đủ 5 giá trị này (theo đúng thứ tự Bandwidth-Delay-Reliability-Load-MTU) để EIGRP có cơ sở tính toán và so sánh route đó với các route EIGRP gốc khác.

## 8. Verify EIGRP

```
Router# show ip eigrp neighbors
Router# show ip eigrp topology
Router# show ip route eigrp
```

`show ip eigrp topology` hiển thị cả Successor lẫn Feasible Successor (nếu có) cho từng đích — lệnh quan trọng nhất để hiểu DUAL đang "nhìn thấy" đường dự phòng nào, khác với `show ip route` chỉ hiển thị Successor đang thực sự dùng.

## 9. Case study — "EIGRP SIA Incident": khi Feasible Successor không tồn tại

> [!example] Case study kỹ thuật — Positive Feedback Failure trong sự cố EIGRP Stuck In Active
> Một bài phân tích sự cố thực tế (rule11.tech) ghi nhận trường hợp EIGRP liên tục mất hội tụ (fail to converge) do hàng loạt neighbor bị "flap" (rớt rồi lên lại) vì sự kiện **Stuck In Active (SIA)** — xảy ra khi router gửi Query hỏi hàng xóm (do không có Feasible Successor sẵn, đúng mục 1) nhưng không nhận được Reply trong thời gian timer cho phép (mặc định 180 giây, cảnh báo SIA-Query ở nửa thời gian). Nguyên nhân gốc trong case này không phải do thiết kế mạng sai, mà do **phân mảnh bộ nhớ (memory fragmentation)** trên router: khi có Query tới, tiến trình EIGRP không tìm được khối bộ nhớ đủ lớn để dựng gói Reply, phải đưa công việc trở lại hàng đợi xử lý sau — trong lúc đó, timer SIA ở router hàng xóm vẫn tiếp tục đếm và cuối cùng hết hạn, làm rớt adjacency, kéo theo hàng loạt Query mới được kích hoạt và lặp lại đúng vấn đề tương tự (positive feedback loop).
>
> **Bài học thực chiến**: sự cố này minh họa rõ vì sao mục 1 nhấn mạnh tầm quan trọng của Feasible Successor — nếu mọi router trong thiết kế đều có sẵn Feasible Successor hợp lệ cho các route quan trọng, phần lớn tình huống Active/Query (và rủi ro SIA đi kèm) có thể tránh được hoàn toàn nhờ chuyển đổi tức thì sang FS mà không cần gửi Query. Thiết kế topology có dự phòng (redundant) với đủ điều kiện Feasibility Condition ở nhiều router là biện pháp phòng ngừa SIA hiệu quả nhất. Nguồn: [rule11.tech — The EIGRP SIA Incident: Positive Feedback Failure in the Wild](https://rule11.tech/the-eigrp-sia-incident-positive-feedback-failure-in-the-wild/).

## 10. So sánh tổng hợp OSPF và EIGRP

| Tiêu chí | OSPF | EIGRP |
|---|---|---|
| Loại giao thức | Link State (mở, IETF) | Hybrid — Advanced Distance Vector (nguồn gốc Cisco) |
| Thuật toán | Dijkstra (SPF) trên LSDB đầy đủ | DUAL, dùng Successor/Feasible Successor |
| Metric | Cost (dựa trên Bandwidth, [[Bai-14_OSPF|Bài 14]] mục 6) | Composite metric (Bandwidth + Delay mặc định, mục 2) |
| Định danh bắt buộc khớp | Area ID phải khớp theo từng liên kết | AS Number phải khớp toàn bộ router muốn làm neighbor (mục 3) |
| Hội tụ khi mất đường chính | Phải tính lại SPF khi topology đổi | Nhanh hơn nếu có sẵn Feasible Successor; chậm (Active/Query) nếu không có |
| Cân bằng tải (load balancing) | Chỉ hỗ trợ equal-cost (các đường cost bằng nhau) | Hỗ trợ cả unequal-cost (dùng `variance`, mục 6), linh hoạt hơn |
| Quảng bá default route | `default-information originate` ([[Bai-14_OSPF|Bài 14]] mục 10) | `redistribute static` từ static default route (mục 7) |
| Khả năng đa hãng | Chuẩn mở, hoạt động tốt giữa nhiều hãng thiết bị | Chủ yếu trong hệ sinh thái Cisco dù đã công bố RFC 7868 |

## 11. Verify & Troubleshoot

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| 2 router không hình thành neighbor dù IP/subnet đúng | AS Number không khớp (mục 3) — không có cảnh báo syslog nào | Đối chiếu `router eigrp <AS>` ở cả 2 đầu, đảm bảo cùng 1 số AS |
| Route biến mất tạm thời khi đường chính hỏng, dù có đường dự phòng vật lý | Đường dự phòng không thỏa Feasibility Condition nên không được công nhận là Feasible Successor — router phải Active/Query, tốn thời gian hơn | `show ip eigrp topology` kiểm tra có Feasible Successor hợp lệ cho route đó hay không |
| Router bị rớt hàng loạt neighbor cùng lúc, log báo SIA | Router hàng xóm không phản hồi Query kịp thời trong 180 giây, có thể do tài nguyên (CPU/bộ nhớ) quá tải (mục 9) | Kiểm tra tải CPU/bộ nhớ router liên quan; thiết kế thêm Feasible Successor để giảm tần suất Query cần gửi |
| Route bị tóm tắt (summary) mất chi tiết subnet ngoài ý muốn | Quên `no auto-summary` trong mạng dùng VLSM (mục 4; chỉ xảy ra trên IOS cũ mặc định bật auto-summary, từ IOS 15.0(1)M mặc định đã tắt) | Thêm `no auto-summary` trên toàn bộ router EIGRP liên quan |
| Đường được chọn không phải đường có bandwidth cao nhất trên toàn tuyến | EIGRP chỉ tính bandwidth **nhỏ nhất** trên đường đi (bottleneck), không phải trung bình (mục 2) | Xác định đúng đoạn có bandwidth thấp nhất trên tuyến bằng `show interfaces`, đối chiếu lại thiết kế nếu cần |
| Chỉ 1 đường được dùng dù đã có 2 Feasible Successor hợp lệ với metric khác nhau | Chưa cấu hình `variance` (mặc định `variance 1` chỉ cho phép equal-cost, mục 6) | Kiểm tra `show ip protocols` xem giá trị variance hiện tại; cấu hình `variance` phù hợp nếu muốn cân bằng tải unequal-cost |
| Router EIGRP khác trong vùng không nhận được default route dù đã `redistribute static` | Quên khai báo đủ 5 giá trị metric sau `redistribute static`, hoặc static default route chưa tồn tại trong routing table nguồn (mục 7) | `show ip route eigrp` kiểm tra route `0.0.0.0/0` đã xuất hiện dạng `D*EX` (external, default route) hay chưa |

## 12. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 3 router hình tam giác (giống Bài 10 mục 1 nhưng cho định tuyến thay vì switching).

1. Cấu hình EIGRP AS 100 trên cả 3 router theo mục 4, xác nhận `show ip eigrp neighbors` thấy đủ hàng xóm liên quan.
2. Chạy `show ip eigrp topology`, xác định Successor và (nếu có) Feasible Successor cho 1 mạng đích cụ thể — ghi lại FD/RD tương ứng.
3. Ngắt liên kết đang là Successor, quan sát tốc độ hội tụ: nếu có Feasible Successor hợp lệ, route chuyển gần như tức thì; nếu không, quan sát trạng thái Active xuất hiện trong `show ip eigrp topology` trước khi hội tụ lại.
4. Cố tình đổi AS Number 1 router thành `200`, quan sát neighbor với router đó biến mất khỏi `show ip eigrp neighbors` mà không có log lỗi rõ ràng — liên hệ mục 3.
5. Cấu hình `variance` theo mục 6 nếu có 2 đường metric khác nhau tới cùng 1 đích, xác nhận cả 2 đường cùng xuất hiện trong `show ip route eigrp` thay vì chỉ 1 đường.
6. Thêm static default route và `redistribute static` theo mục 7 trên 1 router biên, xác nhận các router EIGRP khác nhận được route `0.0.0.0/0` dạng `D*EX`.

**Kết quả kỳ vọng**: phân biệt rõ ràng bằng thực hành sự khác biệt tốc độ hội tụ giữa "có Feasible Successor" và "không có", tự tay tái hiện lỗi AS Number mismatch, và thành thạo cấu hình variance/default route trong EIGRP.

## 13. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| EIGRP luôn hội tụ tức thì trong mọi tình huống nhờ DUAL | Bỏ qua trường hợp không có Feasible Successor khả dụng (mục 1, 9) | Chỉ hội tụ tức thì khi có sẵn Feasible Successor; nếu không, phải qua Active/Query chậm hơn, có rủi ro SIA |
| Process/AS Number của EIGRP chỉ mang tính hình thức như OSPF Process ID | Nhầm lẫn quy tắc giữa 2 giao thức | AS Number EIGRP bắt buộc khớp giữa các router muốn làm neighbor, khác hẳn Process ID OSPF chỉ cục bộ |
| Feasible Successor là bất kỳ đường dự phòng nào tồn tại về mặt vật lý | Bỏ qua điều kiện toán học Feasibility Condition | Chỉ hàng xóm có RD < FD hiện tại mới được công nhận Feasible Successor, đảm bảo loop-free |
| EIGRP tính bandwidth toàn tuyến bằng cách cộng dồn giống Delay | Nhầm lẫn cách tính 2 thành phần khác nhau trong metric | Bandwidth lấy giá trị **nhỏ nhất** trên đường đi (bottleneck), chỉ có Delay mới cộng dồn qua từng hop |
| Advertised Distance và Administrative Distance là cùng 1 khái niệm vì cùng viết tắt "AD" | Nhầm lẫn 2 thuật ngữ trùng chữ viết tắt nhưng khác bản chất hoàn toàn | Advertised Distance = Reported Distance (metric hàng xóm quảng bá, mục 1); Administrative Distance là độ tin cậy nguồn route ([[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 4) |
| `variance` cho phép dùng bất kỳ đường dự phòng nào miễn là muốn chia tải | Bỏ qua điều kiện Feasibility Condition vẫn phải thỏa trước (mục 6) | `variance` chỉ áp dụng cho các route đã là Feasible Successor hợp lệ, không phá vỡ nguyên tắc loop-free của DUAL |

## 14. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Feasible Successor khác Successor ở điểm nào, và vì sao nó giúp EIGRP hội tụ nhanh?**
→ Successor là đường tốt nhất đang thực sự dùng để forward traffic. Feasible Successor là đường dự phòng đã được tính sẵn trước, thỏa Feasibility Condition (RD < FD). Khi Successor hỏng, nếu có Feasible Successor, router chuyển ngay sang dùng nó mà không cần gửi Query hỏi hàng xóm — đây là lý do EIGRP hội tụ nhanh trong trường hợp này.

**2. Feasibility Condition là gì, và nó đảm bảo điều gì về mặt loop?**
→ Điều kiện RD của hàng xóm phải nhỏ hơn FD hiện tại của router. Điều này đảm bảo hàng xóm đó không đi qua chính router hiện tại để tới đích, nên chọn nó làm đường dự phòng sẽ không bao giờ tạo ra loop.

**3. Vì sao 2 router láng giềng cấu hình đúng IP/subnet nhưng vẫn không hình thành EIGRP neighbor?**
→ Khả năng cao nhất là AS Number không khớp giữa 2 router — đây là điều kiện bắt buộc của EIGRP, khác với OSPF. Đáng chú ý là lỗi này không tạo ra bất kỳ cảnh báo syslog nào, cần chủ động đối chiếu cấu hình.

**4. Điều gì xảy ra khi EIGRP không có Feasible Successor và đường chính hỏng?**
→ Router phải chuyển sang trạng thái Active, gửi Query hỏi các hàng xóm liên quan để tìm đường thay thế — quá trình này chậm hơn nhiều so với việc có sẵn Feasible Successor, và nếu hàng xóm không phản hồi kịp trong 180 giây có thể dẫn tới sự kiện Stuck In Active (SIA).

**5. So với OSPF, vì sao EIGRP có thể hỗ trợ cân bằng tải unequal-cost còn OSPF thì không?**
→ EIGRP dùng tham số `variance` cho phép đưa vào routing table cả những đường có metric cao hơn Successor (trong giới hạn hệ số variance) để cùng chia tải, trong khi OSPF theo thiết kế mặc định chỉ đưa vào các đường có cost bằng nhau tuyệt đối (equal-cost) do cách tính SPF của Dijkstra.

**6. Advertised Distance và Reported Distance có phải 2 khái niệm khác nhau không?**
→ Không, đây là 2 tên gọi cho cùng 1 khái niệm — metric mà 1 router hàng xóm quảng bá để tới 1 đích cụ thể. "Advertised Distance (AD)" là thuật ngữ Cisco cũ hơn, "Reported Distance (RD)" là tên gọi phổ biến hơn trong tài liệu hiện đại, dùng để tránh nhầm với Administrative Distance (cũng viết tắt AD nhưng là khái niệm hoàn toàn khác).

**7. EIGRP quảng bá default route vào vùng bằng cách nào, khác OSPF ra sao?**
→ EIGRP không có lệnh tương đương `default-information originate` — phải dùng `redistribute static` để tái phân phối 1 static default route (`0.0.0.0/0`) đã cấu hình sẵn vào EIGRP, kèm khai báo tay đủ 5 giá trị metric (Bandwidth-Delay-Reliability-Load-MTU) vì static route không có sẵn các thành phần này.

**8. EIGRP for IPv6 khác EIGRP IPv4 ở điểm nào về cách kích hoạt trên interface?**
→ EIGRP for IPv6 không dùng lệnh `network` như IPv4 — thay vào đó kích hoạt trực tiếp trên từng interface bằng lệnh `ipv6 eigrp <AS>`, và tiến trình `ipv6 router eigrp` mặc định ở trạng thái shutdown, cần `no shutdown` tường minh mới bắt đầu chạy.

## 15. Tài liệu tham khảo

- [PingLabz — EIGRP DUAL Algorithm: Successor, Feasible Successor, FD, RD](https://www.pinglabz.com/eigrp-dual-algorithm/)
- [PingLabz — EIGRP Metric and K Values: The Formula Explained](https://www.pinglabz.com/eigrp-metric-k-values/)
- [PingLabz — EIGRP Neighbor Requirements: The 5 Things That Must Match](https://www.pinglabz.com/eigrp-neighbor-requirements/)
- [rule11.tech — The EIGRP SIA Incident: Positive Feedback Failure in the Wild](https://rule11.tech/the-eigrp-sia-incident-positive-feedback-failure-in-the-wild/)
- [NetworkLessons — EIGRP Unequal Cost Load Balancing](https://networklessons.com/eigrp/eigrp-unequal-cost-load-balancing)
- [Cisco — Configure Default-Route in EIGRP](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/200279-Configure-Default-route-in-EIGRP.html)
- [NetworkLessons — IPv6 EIGRP on Cisco IOS Router](https://networklessons.com/ipv6/how-to-configure-ipv6-eigrp-on-cisco-ios-router)

---
**Liên kết**: [[Bai-14_OSPF|← Bài 14]] · [[Bai-16_Basic-ACL|Bài 16 - Basic ACL →]]
