---
title: Bài 13 - Static Route
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — Static Route là kỹ năng cấu hình định tuyến đầu tiên phải thành thạo trước khi học OSPF/EIGRP, và Floating Static Route là câu hỏi thực chiến rất hay gặp"
tags: [networking, cisco, routing, static-route, default-route, floating-static, sysadmin]
aliases: [Static Route, Default Route, Floating Static Route]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-12_Inter-VLAN-Routing]]"]
last_review: 2026-07-21
---

# Bài 13: Static Route

> [!important] Tại sao cần file này
> [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] (mục 5) đã giới thiệu Static Route là 1 trong 3 nguồn route, nhưng chưa đi vào chi tiết cách cấu hình và các biến thể của nó. Static Route là kỹ năng định tuyến **cơ bản nhất phải thành thạo trước** khi học các giao thức động (OSPF ở [[Bai-14_OSPF|Bài 14]], EIGRP ở [[Bai-15_EIGRP|Bài 15]]) — không chỉ vì đơn giản hơn, mà vì nhiều khái niệm nền tảng (Administrative Distance, next-hop, recursive lookup) đều xuất phát từ static route và được tái sử dụng xuyên suốt các giao thức động sau này.

## 1. Mạng phù hợp với định tuyến tĩnh, và AD/Metric của Static Route

Static Route phù hợp nhất với các mạng **nhỏ, ít thay đổi topology**, đặc biệt là mạng dạng **stub** (chỉ có đúng 1 đường ra duy nhất, ví dụ chi nhánh nhỏ chỉ nối về 1 trụ sở chính) — nơi việc dùng giao thức động (OSPF/EIGRP) sẽ "quá tay" so với nhu cầu thực tế, chỉ tổ tốn thêm tài nguyên CPU/băng thông (đã so sánh chi tiết ở [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 5). Static Route cũng thường được dùng làm route dự phòng bổ sung ngay cả trong mạng đã chạy giao thức động (mục 5).

Về Administrative Distance, Static Route mặc định có **AD = 1** — chỉ đứng sau Connected (AD = 0), nghĩa là gần như luôn được ưu tiên hơn mọi giao thức động nếu cùng khai báo 1 đích giống hệt nhau (đã học ở [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 4). Static Route **không có khái niệm Metric thực sự** như các giao thức động — giá trị hiển thị trong `[AD/Metric]` của 1 static route (ví dụ `[1/0]`) luôn có phần Metric bằng `0`, vì bản chất route được khai báo tay, không cần "tính toán" chất lượng đường đi.

## 2. Cú pháp cơ bản

```
Router(config)# ip route <mạng đích> <subnet mask> <next-hop hoặc exit-interface>
```

Ví dụ: để router biết đường tới mạng `20.20.20.0/24` qua router hàng xóm có IP `10.10.10.2`:

```
Router(config)# ip route 20.20.20.0 255.255.255.0 10.10.10.2
```

## 3. Ba kiểu khai báo Static Route

| Kiểu | Cú pháp | Đặc điểm |
|---|---|---|
| **Next-hop only** (recursive) | `ip route 20.20.20.0 255.255.255.0 10.10.10.2` | Router phải tra thêm 1 lần nữa trong routing table để tìm ra exit-interface tương ứng với `10.10.10.2` — gọi là **recursive lookup** |
| **Exit-interface only** | `ip route 20.20.20.0 255.255.255.0 GigabitEthernet0/0` | Router gán thẳng route vào 1 interface, không cần tra thêm — chỉ nên dùng cho liên kết point-to-point (ví dụ Serial), **không khuyến nghị cho Ethernet** (multi-access) |
| **Fully specified** | `ip route 20.20.20.0 255.255.255.0 GigabitEthernet0/0 10.10.10.2` | Khai báo cả 2 — vừa tránh recursive lookup, vừa an toàn trên Ethernet |

> [!question] Vì sao "Next-hop only" cần recursive lookup, còn "Fully specified" thì không?
> Với "Next-hop only", router chỉ biết IP `10.10.10.2` — nó phải tra tiếp trong routing table xem IP đó **đi ra interface nào**, tốn thêm 1 bước xử lý mỗi khi forward gói tin (dù không đáng kể về tốc độ với phần cứng hiện đại, nhưng có ảnh hưởng tới hành vi khi mất kết nối — xem mục 5). "Fully specified" đã khai báo sẵn cả interface lẫn next-hop, nên router forward thẳng luôn không cần tra thêm.

> [!question] Vì sao "Exit-interface only" không khuyến nghị dùng trên Ethernet?
> Vì Ethernet là môi trường **multi-access** (nhiều thiết bị cùng chung 1 segment) — chỉ khai báo exit-interface, router không biết chính xác **địa chỉ MAC của next-hop nào** để gửi gói tin tới, buộc phải gửi kèm ARP Request cho **toàn bộ gói tin** đang chờ định tuyến qua route đó, gây lãng phí tài nguyên không cần thiết. Trên liên kết point-to-point (chỉ có đúng 2 đầu, ví dụ Serial), vấn đề này không tồn tại vì chỉ có duy nhất 1 thiết bị ở đầu kia.

## 4. Default Route

**Default Route** (`0.0.0.0/0`) là route "cuối cùng" — router dùng route này khi **không tìm thấy bất kỳ route nào khác khớp hơn** với địa chỉ đích (đúng theo nguyên tắc Longest Prefix Match đã học ở [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 2, vì `/0` luôn là prefix ngắn nhất có thể, chỉ được chọn khi không còn route nào cụ thể hơn).

```
Default route ra Internet:
+------------+    +------------+
|     R1     |----|    ISP     |
|   LAN-A    |    | (Internet) |
+------------+    +------------+
        S* 0.0.0.0/0 [1/0] via ISP
```

```
Router(config)# ip route 0.0.0.0 0.0.0.0 10.10.10.2
```

Đây là cách cấu hình phổ biến nhất cho router biên (edge router) kết nối ra ISP — thay vì phải khai báo route tới từng dải IP trên toàn bộ Internet (điều không tưởng), router chỉ cần biết "nếu không phải mạng nội bộ, cứ đẩy hết ra hướng ISP".

## 5. Floating Static Route

**Floating Static Route** là static route được cấu hình với **Administrative Distance cao hơn** route chính (route chính có thể là route động như OSPF/EIGRP, hoặc 1 static route khác) — dùng làm **đường dự phòng (backup)**, chỉ được đưa vào routing table khi route chính biến mất.

```
+----------+                              +----------+
|    R1    |===== duong chinh (AD 1) =====|    R2    |
|          |----- duong backup (AD 5) ----|          |
+----------+                              +----------+
```

```
Router(config)# ip route 20.20.20.0 255.255.255.0 10.10.10.2
Router(config)# ip route 20.20.20.0 255.255.255.0 10.10.20.2 5
```

Số `5` ở cuối lệnh thứ hai là AD tùy chỉnh (mặc định static route là AD 1) — vì `5 > 1`, route thứ hai chỉ được dùng khi route đầu tiên **không còn tồn tại** trong routing table (ví dụ đường chính đứt cáp).

> [!question] Vì sao Floating Static Route "backup cho OSPF" thường đặt AD lớn hơn 110 (phổ biến 120-254 tùy thiết kế), còn "backup cho EIGRP internal" thường đặt AD trong khoảng 91-109?
> Vì AD của route backup phải **cao hơn** AD của route chính nó backup, nhưng đồng thời vẫn nên **thấp hơn** AD của các giao thức "kém tin cậy hơn" khác nếu muốn giữ đúng thứ tự ưu tiên tổng thể. OSPF có AD mặc định 110, nên floating static backup cho OSPF cần AD > 110 (thường chọn giá trị ngay trên 110, phổ biến 120-254 tùy thiết kế; nên tránh đúng 200 vì trùng AD của iBGP, dễ xung đột thứ tự ưu tiên). EIGRP internal có AD 90, nên floating static backup cho nó thường chọn khoảng 91-109 — vẫn cao hơn 90 nhưng thấp hơn AD 110 của OSPF, tránh xung đột thứ tự ưu tiên nếu mạng có chạy đồng thời nhiều giao thức.

> [!question] Floating Static Route dùng "Next-hop only" (recursive) có rủi ro gì khi đường chính bị đứt?
> Đây là 1 lỗi thiết kế thực chiến dễ gặp: nếu next-hop của floating static vẫn **còn có thể tới được bằng đường vòng khác** (dù đường vòng đó dài/chậm hơn nhiều) sau khi đường chính đứt, route recursive **không bị rút khỏi routing table** — router vẫn cố dùng route đó, có thể khiến traffic đi vòng vèo không kiểm soát hoặc gây loop tạm thời. Route "Fully specified" (gắn thẳng với 1 interface cụ thể) sẽ tự động bị rút ngay khi interface đó down — hành vi rõ ràng và an toàn hơn nhiều cho floating static route dùng làm backup.

## 6. Static Route cho IPv6

Cú pháp static route cho IPv6 (đã học địa chỉ IPv6 ở [[Bai-06_Dia-chi-IPv6|Bài 6]]) tương tự IPv4 về nguyên tắc, nhưng dùng lệnh riêng `ipv6 route` với prefix dạng CIDR (không có subnet mask rời như IPv4):

```
Router(config)# ipv6 route <IPv6 prefix>/<độ dài prefix> <next-hop hoặc exit-interface>
```

Ví dụ, để router biết đường tới mạng `2001:db8:20:20::/64` qua next-hop `2001:db8:10:10::2`:

```
Router(config)# ipv6 route 2001:db8:20:20::/64 2001:db8:10:10::2
```

Default route IPv6 (tương đương `0.0.0.0/0` của IPv4) là `::/0`:

```
Router(config)# ipv6 route ::/0 2001:db8:10:10::2
```

Floating Static Route cho IPv6 cũng dùng đúng cơ chế thêm AD tùy chỉnh ở cuối lệnh, giống hệt IPv4:

```
Router(config)# ipv6 route 2001:db8:20:20::/64 2001:db8:10:20::2 5
```

> [!question] Vì sao lệnh `ipv6 route` không có tham số subnet mask rời như `ip route` của IPv4?
> Vì IPv6 chuẩn hóa cách viết địa chỉ luôn đi kèm độ dài prefix dạng CIDR (`/64`, `/48`...) ngay từ thiết kế ban đầu — khác với IPv4 vốn có lịch sử dùng subnet mask dạng số thập phân chấm (`255.255.255.0`) từ thời kỳ trước khi CIDR ra đời (đã học ở [[Bai-05_Dia-chi-IPv4|Bài 5]]). IPv6 không cần "gánh" cách viết cũ này, nên cú pháp gọn hơn hẳn.

Trước khi cấu hình bất kỳ route IPv6 nào, router cần bật định tuyến IPv6 toàn cục:

```
Router(config)# ipv6 unicast-routing
```

## 7. Verify & Troubleshoot

| Lệnh | Công dụng |
|---|---|
| `show ip route static` | Chỉ hiển thị các route static trong bảng, dễ rà soát hơn `show ip route` đầy đủ |
| `show ip route 20.20.20.0` | Xem chi tiết đúng 1 route cụ thể, gồm cả AD/Metric |
| `show running-config \| include ip route` | Xem toàn bộ các lệnh `ip route` đã cấu hình trong running-config |

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Static route cấu hình xong nhưng không xuất hiện trong `show ip route` | Next-hop khai báo không thể "resolve" được (không có route nào dẫn tới chính next-hop đó) | Kiểm tra `show ip route <next-hop>` xem router có biết đường tới next-hop hay không trước |
| Đã cấu hình static route đúng nhưng router vẫn dùng route khác (route động) cho cùng đích | Route static có AD cao hơn route động đang tồn tại cho cùng đích (ít gặp vì static AD=1 mặc định thấp, nhưng có thể do đã chỉnh tay AD static cao hơn) | `show ip route <đích>` xem AD của route đang được chọn, đối chiếu lại AD đã đặt cho static |
| Floating static không kích hoạt dù đường chính đã đứt | Route chính vẫn còn "resolve" được qua 1 đường khác dù chậm hơn (mục 5, recursive lookup) | Ưu tiên dùng Fully specified static thay vì Next-hop only cho floating static để đảm bảo rút route ngay khi interface down |
| Default route hoạt động nhưng vẫn có 1 số IP nội bộ không ra được Internet | Thiếu NAT (chưa học, thuộc phạm vi bài sau) hoặc thiếu route cụ thể hơn cho 1 số subnet nội bộ khiến traffic đi nhầm ra default route | `show ip route <IP đích>` xác nhận route nào thực sự được chọn theo Longest Prefix Match |

## 8. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 2 router nối 2 đường (1 chính, 1 dự phòng) tới cùng 1 mạng đích.

1. Cấu hình static route "Fully specified" theo mục 3 giữa 2 router, xác nhận 2 mạng ở 2 đầu ping thông nhau.
2. Thêm route `0.0.0.0 0.0.0.0` (default route) trên router biên trỏ ra 1 router đóng vai trò ISP giả lập, xác nhận PC nội bộ ping thông IP "ngoài Internet" giả lập đó.
3. Cấu hình Floating Static Route theo mục 5 (AD 5) cho đường dự phòng, xác nhận `show ip route` chỉ hiện đường chính (AD 1).
4. `shutdown` interface của đường chính, quan sát floating static tự động xuất hiện trong `show ip route` thay thế — xác nhận traffic vẫn đi được qua đường dự phòng.
5. Lặp lại bước 1-2 với địa chỉ IPv6 theo mục 6 (`ipv6 unicast-routing`, `ipv6 route`), xác nhận PC IPv6 ping thông nhau qua static route IPv6.

**Kết quả kỳ vọng**: thành thạo cả 3 kiểu khai báo static route (cả IPv4 lẫn IPv6), hiểu rõ cơ chế AD quyết định route nào được chọn, và tự tay chứng minh được cơ chế failover của Floating Static Route.

## 9. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Cứ khai báo `ip route` là đủ, không cần quan tâm dùng next-hop hay exit-interface | Bỏ qua khác biệt hành vi ARP và recursive lookup giữa các kiểu (mục 3) | Trên Ethernet nên ưu tiên Fully specified hoặc Next-hop only; Exit-interface only chỉ phù hợp point-to-point |
| Default route (`0.0.0.0/0`) luôn được ưu tiên vì "là route mặc định" | Nhầm "default" với "được ưu tiên cao" | Default route có prefix `/0` ngắn nhất, theo Longest Prefix Match nó **luôn bị các route cụ thể hơn** ghi đè, chỉ dùng khi không còn route nào khác khớp |
| Floating Static Route tự động rút khỏi bảng ngay khi đường chính có vấn đề, bất kể kiểu khai báo nào | Bỏ qua rủi ro recursive lookup ở static route dùng next-hop only (mục 5) | Chỉ Fully specified static mới đảm bảo rút route ngay lập tức khi interface down; Next-hop only có thể "trơ" nếu next-hop vẫn resolve được qua đường khác |
| AD của floating static càng cao càng tốt, không quan trọng chọn số bao nhiêu | Bỏ qua việc AD cần nằm đúng vị trí trong thứ tự ưu tiên tổng thể của mạng | AD floating static cần cao hơn route chính nhưng vẫn nên hợp lý so với các giao thức khác đang chạy trong mạng (mục 5) |
| Static route IPv6 dùng chung cú pháp `ip route` với IPv4, chỉ đổi định dạng địa chỉ | Nhầm lẫn 2 lệnh cấu hình hoàn toàn khác nhau | IPv6 dùng lệnh riêng `ipv6 route`, và bắt buộc phải bật `ipv6 unicast-routing` trước (mục 6) |

## 10. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Recursive lookup trong static route là gì, và vì sao "Fully specified" tránh được nó?**
→ Recursive lookup xảy ra khi static route chỉ khai báo next-hop IP, buộc router phải tra thêm 1 lần trong routing table để tìm ra exit-interface tương ứng. "Fully specified" khai báo sẵn cả interface lẫn next-hop nên không cần bước tra thêm này.

**2. Default route là gì, và vì sao nó luôn có mức ưu tiên thấp nhất khi so khớp địa chỉ đích?**
→ Default route (`0.0.0.0/0`) là route dùng khi không có route nào khác khớp với đích. Vì `/0` là prefix ngắn nhất có thể (không cụ thể), theo nguyên tắc Longest Prefix Match nó luôn bị bất kỳ route cụ thể hơn nào ghi đè, chỉ được chọn khi không còn lựa chọn nào khác.

**3. Floating Static Route hoạt động dựa trên cơ chế nào?**
→ Dựa trên Administrative Distance: route backup được đặt AD cao hơn route chính, nên chỉ được router đưa vào routing table khi route chính (AD thấp hơn) không còn tồn tại — ví dụ khi route động bị mất do đứt liên kết.

**4. Vì sao không nên dùng "Exit-interface only" cho static route trên cổng Ethernet?**
→ Vì Ethernet là môi trường multi-access, router không xác định được chính xác MAC của thiết bị next-hop nào để gửi gói tin, dẫn đến phải ARP cho toàn bộ gói tin chờ định tuyến qua route đó, gây lãng phí xử lý không cần thiết — chỉ phù hợp trên liên kết point-to-point như Serial.

**5. Nếu dùng Floating Static Route kiểu "Next-hop only" và đường chính bị đứt nhưng next-hop vẫn "resolve" được qua đường khác, điều gì xảy ra?**
→ Route floating static đó sẽ **không** tự động được rút khỏi routing table dù đường chính đã hỏng, vì bản chất recursive vẫn còn resolve được next-hop — có thể khiến traffic đi vòng không kiểm soát thay vì chuyển hẳn sang đường dự phòng dự kiến, đây là lý do nên ưu tiên Fully specified cho floating static.

**6. Static route IPv6 khác static route IPv4 ở điểm nào về cú pháp?**
→ IPv6 dùng lệnh riêng `ipv6 route <prefix>/<độ dài prefix> <next-hop>`, không có subnet mask rời như `ip route` của IPv4 vì IPv6 chuẩn hóa viết theo CIDR ngay từ đầu. Ngoài ra bắt buộc phải bật `ipv6 unicast-routing` trước khi cấu hình bất kỳ route IPv6 nào, khác với IPv4 vốn có định tuyến bật sẵn mặc định.

## 11. Tài liệu tham khảo

- [Cisco — Configure a Next Hop IP Address for Static Routes](https://www.cisco.com/c/en/us/support/docs/dial-access/floating-static-route/118263-technote-nexthop-00.html)
- [NetsTuts — Floating Static Routes: Backup Routes, AD & Failover](https://www.netstuts.com/floating-static-routes)
- [PingLabz — Static Routing on Cisco IOS XE: Default & Floating](https://www.pinglabz.com/static-routing-cisco-ios-xe/)
- [ComputingForGeeks — Configure Cisco IPv4 Static Routes & Floating](https://computingforgeeks.com/cisco-ipv4-static-routes-configuration/)
- [Cisco — IP Routing Configuration Guide: IPv6 Routing - Static Routing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_ip6-route-static-xe.html)

---
**Liên kết**: [[Bai-12_Inter-VLAN-Routing|← Bài 12]] · [[Bai-14_OSPF|Bài 14 - OSPF →]]
