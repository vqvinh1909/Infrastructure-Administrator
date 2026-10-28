---
title: Bài 23 - Network Load Balancing
type: Concept
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — Load Balancing và PBR là kiến thức thường được hỏi khi thiết kế mạng có nhiều đường ra Internet hoặc nhiều server cùng dịch vụ"
tags: [networking, cisco, load-balancing, pbr, ecmp, sysadmin]
aliases: [Load Balancing, Cân bằng tải, PBR, Policy Based Routing, ECMP]
related: ["[[Bai-10_STP-va-EtherChannel]]", "[[Bai-21_FHRP]]"]
last_review: 2026-07-21
---

# Bài 23: Network Load Balancing

> [!important] Tại sao cần file này
> Các bài trước đã học nhiều kỹ thuật dự phòng riêng lẻ: EtherChannel ([[Bai-10_STP-va-EtherChannel|Bài 10]]) gộp nhiều liên kết Lớp 2, GLBP ([[Bai-21_FHRP|Bài 21]]) chia tải Gateway Lớp 3. Bài này hệ thống hóa lại toàn bộ khái niệm **Load Balancing** theo từng tầng OSI, đồng thời giới thiệu **Policy Based Routing (PBR)** — công cụ định tuyến traffic theo policy tùy chỉnh, vượt ra ngoài khả năng của routing table thông thường đã học ở Module 3.

## 1. Sự cần thiết của cân bằng tải

Nếu toàn bộ traffic chỉ đi qua **1 đường/1 thiết bị duy nhất**, hệ thống đó vừa là điểm nghẽn hiệu năng (bottleneck) vừa là điểm lỗi đơn (Single Point of Failure — đã nhắc tới ở case study Facebook 2021 trong [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]]). Load Balancing giải quyết đồng thời 2 vấn đề: **phân phối tải** ra nhiều đường/thiết bị để tránh quá tải cục bộ, và **tăng khả năng chịu lỗi** — nếu 1 trong số nhiều đường/thiết bị đó hỏng, phần còn lại vẫn tiếp tục phục vụ.

## 2. Thiết bị và dịch vụ hỗ trợ Load Balancing

| Loại | Ví dụ | Vai trò |
|---|---|---|
| Tính năng có sẵn trên Switch/Router | EtherChannel ([[Bai-10_STP-va-EtherChannel|Bài 10]]), GLBP ([[Bai-21_FHRP|Bài 21]]), ECMP (định tuyến động, mục 3) | Cân bằng tải ở tầng hạ tầng mạng, không cần thiết bị chuyên dụng riêng |
| Load Balancer phần cứng chuyên dụng | F5 BIG-IP, Citrix ADC (NetScaler) | Cân bằng tải cho ứng dụng/dịch vụ ở quy mô lớn, nhiều tính năng nâng cao (health check, SSL offload) |
| Load Balancer phần mềm | HAProxy, Nginx | Giải pháp chi phí thấp hơn, linh hoạt triển khai trên server thông thường hoặc cloud |
| Tính năng định tuyến tùy chỉnh | Policy Based Routing (mục 4) | Không cân bằng tải theo nghĩa "chia đều", mà định tuyến traffic theo policy cụ thể (ví dụ traffic từ phòng ban A luôn đi ra ISP 1) |

## 3. Kỹ thuật cân bằng tải theo tầng OSI

**Lớp 2 — EtherChannel**: đã học chi tiết ở [[Bai-10_STP-va-EtherChannel|Bài 10]] — gộp nhiều liên kết vật lý thành 1 liên kết logic, phân bổ traffic dựa trên thuật toán băm (hash) theo MAC/IP nguồn-đích.

**Lớp 3 — ECMP (Equal-Cost Multi-Path)**: khi routing table có **nhiều route cùng đích, cùng metric** (đã học AD/Metric ở [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 4), router tự động cân bằng tải giữa các đường đó — cơ chế này xảy ra tự nhiên với OSPF/EIGRP khi thiết kế mạng có nhiều đường cost bằng nhau, không cần cấu hình thêm gì đặc biệt.

**Lớp 4 — Load Balancer theo Port (TCP/UDP)**:

```
Layer 4 Load Balancer phan phoi traffic toi 3 server:
            +--------------------+
            |   LOAD BALANCER    |
            | (VIP: 20.20.20.1)  |
            +--------------------+
                       |
            ----------------------
+------------+  +------------+  +------------+
|  SERVER 1  |  |  SERVER 2  |  |  SERVER 3  |
|            |  |            |  |            |
+------------+  +------------+  +------------+
```

Load Balancer đứng trước 1 nhóm server (server farm), quảng bá **1 địa chỉ IP ảo (VIP)** duy nhất ra ngoài — người dùng chỉ biết VIP đó, không biết có bao nhiêu server thật đứng sau. Load Balancer phân phối kết nối tới từng server theo 1 trong các thuật toán:

| Thuật toán | Cơ chế |
|---|---|
| **Round Robin** | Lần lượt phân phối kết nối mới theo vòng tròn qua từng server |
| **Least Connection** | Ưu tiên gửi kết nối mới tới server đang có ít kết nối đang xử lý nhất |
| **IP Hash** | Băm địa chỉ IP nguồn để luôn gửi cùng 1 Client tới cùng 1 server (duy trì session) |

> [!question] Vì sao ECMP (Lớp 3) không cần cấu hình gì thêm nhưng Load Balancer chuyên dụng (Lớp 4) lại cần thiết bị/tính năng riêng?
> Vì ECMP chỉ đơn thuần cân bằng tải theo **route/đường đi**, không quan tâm nội dung traffic là gì — router chỉ cần thấy nhiều route cùng cost là tự động chia tải theo flow (dựa trên hash IP nguồn-đích). Load Balancer Lớp 4 phải hiểu khái niệm "kết nối" (connection) ở tầng Transport, theo dõi trạng thái từng kết nối để quyết định gửi tới server nào theo đúng thuật toán yêu cầu (ví dụ Least Connection cần biết chính xác server nào đang bận) — đây là chức năng phức tạp hơn nhiều so với việc router chỉ nhìn vào routing table.

## 4. Triển khai Policy Based Routing (PBR)

PBR cho phép định tuyến traffic **không theo routing table thông thường**, mà theo **policy tùy chỉnh** (dựa trên nguồn, ứng dụng, hoặc bất kỳ tiêu chí nào ACL hỗ trợ) — dùng phổ biến để chủ động phân luồng traffic qua nhiều đường ra Internet (ví dụ 2 ISP) theo tiêu chí riêng thay vì để router tự chọn theo AD/Metric.

```
Router(config)# access-list 10 permit 10.10.20.0 0.0.0.255
Router(config)# route-map PBR-ISP1 permit 10
Router(config-route-map)# match ip address 10
Router(config-route-map)# set ip next-hop 203.0.113.1
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ip policy route-map PBR-ISP1
```

Cấu hình trên buộc **toàn bộ traffic có nguồn từ mạng `10.10.20.0/24`** (ví dụ phòng Kinh doanh) đi ra qua next-hop `203.0.113.1` (ISP 1), bất kể routing table thông thường đang khuyến nghị đường nào khác — trong khi traffic từ các mạng khác vẫn định tuyến bình thường theo routing table.

> [!question] Vì sao PBR được xem là "cân bằng tải" dù về bản chất nó chỉ ép buộc 1 luồng traffic cụ thể đi 1 đường cố định?
> Vì khi áp dụng PBR cho **nhiều nhóm traffic khác nhau, mỗi nhóm trỏ 1 đường khác nhau** (ví dụ phòng Kinh doanh qua ISP 1, phòng Kỹ thuật qua ISP 2), kết quả tổng thể là tải được **phân bổ chủ động** giữa nhiều đường ra thay vì dồn hết vào 1 đường duy nhất — dù cơ chế phân bổ là "theo policy cố định" (deterministic) chứ không "tự động chia đều" như ECMP hay Load Balancer thật sự. Đây là lý do PBR thường được xếp vào nhóm kỹ thuật load balancing "thủ công", phù hợp khi cần kiểm soát chính xác luồng traffic nào đi đường nào (ví dụ vì lý do chi phí băng thông hoặc SLA khác nhau giữa các ISP).

## 5. Kiểm tra dự phòng hệ thống

Việc triển khai load balancing chỉ có ý nghĩa nếu dự phòng **thực sự hoạt động khi có sự cố** — cần chủ động kiểm tra định kỳ, không chỉ tin tưởng cấu hình đã đúng lúc ban đầu:

```
Router# show etherchannel summary
Router# show ip route
Router# show route-map
Router# show ip policy
```

Quy trình kiểm tra dự phòng chuẩn: cố tình ngắt **từng thành phần một** (rút 1 cáp trong EtherChannel, tắt 1 server sau Load Balancer, tắt 1 đường ISP) trong môi trường bảo trì đã lên kế hoạch, xác nhận hệ thống vẫn hoạt động bình thường qua thành phần còn lại — đây là cách duy nhất để biết chắc chắn dự phòng "trên giấy" có thực sự hoạt động "trong thực tế" hay không.

## 6. Verify & Troubleshoot

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| PBR cấu hình đúng nhưng traffic vẫn đi theo routing table thông thường | Quên `ip policy route-map` trên đúng interface, hoặc ACL trong route-map không khớp traffic thực tế | `show ip policy` xác nhận route-map đã gán đúng interface; đối chiếu lại ACL |
| Traffic dồn hết vào 1 server dù đã có Load Balancer với nhiều server | Thuật toán IP Hash khiến cùng 1 dải IP nguồn luôn về đúng 1 server (đúng thiết kế, không phải lỗi) nếu phần lớn traffic tới từ ít nguồn | Cân nhắc đổi thuật toán (Round Robin/Least Connection) nếu muốn phân bổ đều hơn bất kể nguồn traffic |
| ECMP không tự cân bằng tải dù có nhiều route cùng đích | Các route không thực sự cùng Metric (dù cùng AD), hoặc chỉ có 1 route đang active do lỗi ở đường còn lại | `show ip route` đối chiếu Metric các route, xác nhận cả 2/nhiều đường đều đang up |
| Sau khi test dự phòng, hệ thống không tự phục hồi khi thành phần lỗi được khôi phục | Thiếu cấu hình hoặc thời gian hội tụ lại chưa đủ, hoặc thiết bị dự phòng cần thao tác tay để rejoin | Kiểm tra lại theo đúng giao thức cụ thể đang dùng (EtherChannel/GLBP/định tuyến động) xem có cần preempt hoặc rejoin thủ công |

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 1 router có 2 đường ra "Internet" giả lập (2 ISP), 2 mạng nguồn nội bộ khác nhau.

1. Cấu hình PBR theo mục 4, đảm bảo mạng nguồn A đi ra đúng ISP 1, mạng nguồn B đi ra đúng ISP 2 (dù routing table mặc định có thể chọn khác).
2. Dùng `traceroute` từ PC ở mỗi mạng nguồn, xác nhận traffic thực sự đi đúng ISP đã policy.
3. Tắt đường ISP 1, quan sát traffic từ mạng nguồn A (theo PBR trỏ cứng ISP 1) — xác nhận hành vi khi đường bị policy trỏ tới không còn khả dụng (PBR mặc định không tự failover trừ khi cấu hình thêm cơ chế dự phòng).

**Kết quả kỳ vọng**: hiểu rõ khác biệt giữa cân bằng tải "tự động" (ECMP, EtherChannel) và "theo policy cố định" (PBR), đồng thời nhận ra giới hạn của PBR khi đường được chỉ định gặp sự cố.

## 8. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| PBR tự động chuyển traffic sang đường khác khi đường được chỉ định gặp sự cố | Nhầm PBR với cơ chế dự phòng tự động | PBR mặc định chỉ trỏ cứng theo policy, cần cấu hình thêm cơ chế theo dõi (như `set ip next-hop verify-availability`) mới có khả năng chuyển hướng khi đường đó lỗi |
| ECMP luôn chia đúng 50/50 traffic giữa các đường cùng cost | Nhầm "cân bằng tải" với "chia đều tuyệt đối" | ECMP chia theo flow (dựa trên hash), không đảm bảo chia đều tuyệt đối lưu lượng thực tế nếu phân bổ flow không đồng đều |
| Load Balancing luôn cần thiết bị phần cứng chuyên dụng đắt tiền | Bỏ qua các kỹ thuật có sẵn miễn phí trên router/switch | Nhiều kỹ thuật (EtherChannel, ECMP, GLBP, PBR) đã có sẵn trên thiết bị Cisco thông thường, không cần đầu tư thêm |
| Round Robin luôn là thuật toán tốt nhất cho mọi trường hợp | Bỏ qua đặc thù khác nhau giữa các loại traffic/ứng dụng | Least Connection phù hợp hơn khi các kết nối có thời lượng xử lý chênh lệch lớn; IP Hash cần thiết khi ứng dụng yêu cầu duy trì session về đúng 1 server |

## 9. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao hệ thống cần Load Balancing thay vì chỉ dùng 1 đường/1 thiết bị mạnh nhất có thể?**
→ Vì 1 đường/thiết bị duy nhất vừa là điểm nghẽn hiệu năng vừa là điểm lỗi đơn (Single Point of Failure) — Load Balancing giải quyết cả 2 vấn đề bằng cách phân phối tải và tăng khả năng chịu lỗi khi 1 thành phần gặp sự cố.

**2. ECMP hoạt động dựa trên điều kiện gì trong routing table?**
→ ECMP kích hoạt khi routing table có nhiều route cùng đích với cùng Metric — router tự động cân bằng tải giữa các đường đó theo hash, không cần cấu hình thêm.

**3. PBR khác gì so với định tuyến thông thường dựa trên routing table?**
→ PBR định tuyến traffic theo policy tùy chỉnh (dựa trên ACL match nguồn/đích/ứng dụng), có thể ghi đè quyết định mà routing table thông thường sẽ đưa ra — hữu ích khi cần kiểm soát chính xác luồng traffic nào đi đường nào.

**4. Phân biệt 3 thuật toán Round Robin, Least Connection, và IP Hash trong Load Balancer Lớp 4.**
→ Round Robin phân phối lần lượt theo vòng tròn. Least Connection ưu tiên server đang ít kết nối nhất. IP Hash băm địa chỉ IP nguồn để đảm bảo cùng 1 Client luôn về đúng 1 server, phục vụ duy trì session.

**5. Vì sao cần chủ động test dự phòng (rút cáp/tắt thiết bị) thay vì chỉ tin tưởng cấu hình đã đúng?**
→ Vì cấu hình đúng cú pháp không đảm bảo hành vi thực tế khi sự cố xảy ra đúng như kỳ vọng — chỉ có cách chủ động mô phỏng sự cố trong môi trường bảo trì đã lên kế hoạch mới xác nhận được chắc chắn dự phòng hoạt động đúng.

## 10. Tài liệu tham khảo

- [Kloia — Understanding Load Balancing Across OSI Layers](https://www.kloia.com/blog/understanding-load-balancing-across-osi-layers-layer-3-layer-4-and-layer-7)
- [Cisco — ECMP Load Balancing](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mp_l3_vpns/configuration/xe-3s/asr903/mp-l3-vpns-xe-3s-asr903-book/mp-l3-vpns-xe-3s-asr903-book_chapter_0100.pdf)
- [Cisco — Configure Policy-based Routing with Next-Hop Commands](https://www.cisco.com/c/en/us/support/docs/ip/ip-routed-protocols/47121-pbr-cmds-ce.html)

---
**Liên kết**: [[Bai-22_Remote-Access-VPN|← Bài 22]] · [[Bai-24_Wireless-LAN|Bài 24 - Wireless LAN →]]
