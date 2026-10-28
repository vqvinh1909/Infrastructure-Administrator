---
title: Bài 21 - FHRP (First Hop Redundancy Protocol)
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — FHRP là kỹ thuật dự phòng Gateway bắt buộc trong mọi thiết kế mạng doanh nghiệp có 2 router/switch Layer 3 trở lên ở lớp Distribution"
tags: [networking, cisco, fhrp, hsrp, vrrp, glbp, sysadmin]
aliases: [FHRP, HSRP, VRRP, GLBP, Gateway Redundancy]
related: ["[[Bai-10_STP-va-EtherChannel]]", "[[Bai-12_Inter-VLAN-Routing]]"]
last_review: 2026-07-21
---

# Bài 21: FHRP (First Hop Redundancy Protocol)

> [!important] Tại sao cần file này
> [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]] (mục 5) đã đặt ra nguyên tắc "lớp Distribution luôn cần 2 thiết bị dự phòng nhau", và [[Bai-10_STP-va-EtherChannel|Bài 10]] giải quyết dự phòng ở **Lớp 2** (STP chống loop). Nhưng PC chỉ khai báo được **1 địa chỉ Gateway duy nhất** trong cấu hình IP của nó — nếu router/switch đóng vai trò Gateway đó chết, PC sẽ không tự động biết chuyển sang dùng thiết bị dự phòng còn lại, dù đường vật lý vẫn còn nguyên. **FHRP** giải quyết đúng khoảng trống này: cho phép nhiều router **giả lập chung 1 Gateway ảo duy nhất**, PC không hề biết (và không cần biết) đằng sau nó thực ra có 2+ thiết bị vật lý.

## 1. Chức năng dự phòng Gateway — HSRP, VRRP, GLBP

```
+----------------+     +----------------+
|       R1       |     |       R2       |
|     ACTIVE     |     |    STANDBY     |
|  priority 150  |     |  priority 100  |
+----------------+     +----------------+

VIP dung chung: 10.10.10.1 (0000.0c07.ac01)

         +--------------------+
         |         PC         |
         |  GW = 10.10.10.1   |
         |                    |
         +--------------------+
```

PC chỉ cấu hình đúng 1 Gateway (`10.10.10.1`) — đây là **địa chỉ IP ảo (Virtual IP)** đi kèm 1 **địa chỉ MAC ảo (Virtual MAC)**, không thuộc về riêng R1 hay R2. Cả 2 router chạy 1 giao thức FHRP để tự thống nhất ai đang thực sự trả lời traffic gửi tới Gateway ảo đó — nếu router đang đảm nhận vai trò đó chết, router còn lại tự động tiếp quản **VIP và VMAC y hệt**, PC hoàn toàn không nhận ra có thay đổi (không cần ARP lại).

| Giao thức | Chuẩn | Đặc điểm |
|---|---|---|
| **HSRP** (Hot Standby Router Protocol) | Độc quyền Cisco | 1 Active + 1 Standby, các router còn lại (nếu có) ở trạng thái Listen |
| **VRRP** (Virtual Router Redundancy Protocol) | Mở, IETF (RFC) | 1 Master + nhiều Backup, khái niệm tương đương HSRP nhưng đa hãng |
| **GLBP** (Gateway Load Balancing Protocol) | Độc quyền Cisco | Nhiều router cùng **active đồng thời**, chia tải thực sự thay vì chỉ 1 router xử lý traffic tại 1 thời điểm |

> [!question] Vì sao HSRP/VRRP chỉ gọi là "dự phòng" còn GLBP mới thực sự là "cân bằng tải"?
> Vì với HSRP/VRRP, tại bất kỳ thời điểm nào cũng chỉ có **đúng 1 router** (Active/Master) thực sự xử lý traffic — router còn lại (Standby/Backup) hoàn toàn "rảnh rỗi" về mặt định tuyến, chỉ chờ router kia chết mới tiếp quản. Tài nguyên của Standby/Backup gần như bị lãng phí trong điều kiện bình thường. GLBP khắc phục điều này bằng cách để **nhiều router cùng ở trạng thái forwarding traffic thật sự cùng lúc** (chi tiết cơ chế AVG/AVF ở mục 4) — tận dụng được băng thông của cả 2 (hoặc nhiều) thiết bị thay vì chỉ 1.

## 2. Phương thức hoạt động của HSRP

Các router trong cùng 1 nhóm HSRP trao đổi Hello message (mặc định mỗi **3 giây**, Hold time **10 giây**) để theo dõi lẫn nhau. Router có **Priority cao nhất** (mặc định Priority = 100 trên mọi router) trở thành **Active** — router xử lý toàn bộ traffic gửi tới VIP. Nếu 2 router cùng Priority, router có **IP vật lý cao hơn** thắng.

Virtual MAC của HSRP có dạng cố định: `0000.0c07.ac<XX>`, trong đó `XX` là số nhóm HSRP (group number) viết dưới dạng hex — ví dụ group 1 sẽ có VMAC `0000.0c07.ac01`.

## 3. Active Router, Standby Router, Preempt, Priority

| Khái niệm | Ý nghĩa |
|---|---|
| **Active Router** | Router đang thực sự xử lý traffic gửi tới Gateway ảo |
| **Standby Router** | Router dự phòng, sẵn sàng tiếp quản ngay khi Active router mất tín hiệu (hết Hold time) |
| **Priority** | Giá trị 0-255 quyết định ai làm Active, càng cao càng ưu tiên (mặc định 100) |
| **Preempt** | Cho phép 1 router có Priority cao hơn **giành lại** vai trò Active ngay khi nó phục hồi, thay vì để router Priority thấp hơn tiếp tục giữ vai trò Active |
| **Non-preempt** | Trạng thái **mặc định của HSRP** — router có Priority cao hơn khi phục hồi **không** tự động giành lại vai trò Active, mà chỉ đứng ở Standby chờ router Active hiện tại (dù Priority thấp hơn) gặp sự cố mới được tiếp quản |

> [!question] Vì sao Preempt **mặc định bị tắt** trên HSRP, trong khi VRRP lại **mặc định bật**?
> Đây là khác biệt thiết kế triết lý giữa 2 giao thức: HSRP mặc định tắt Preempt để tránh tình huống router vừa phục hồi sau sự cố (có thể chưa hoàn toàn ổn định — vừa reload, đang nạp lại cấu hình/route) lập tức giành lại vai trò Active, gây gián đoạn traffic 1 lần nữa ngay sau khi vừa ổn định trở lại với router Standby. VRRP theo tinh thần chuẩn mở lại ưu tiên "đúng theo Priority đã cấu hình" ngay khi có thể, nên mặc định bật Preempt. Trong thực tế vận hành HSRP, quản trị viên **chủ động bật `standby preempt`** nếu muốn đảm bảo router chính luôn là Active bất cứ khi nào nó sẵn sàng.

## 4. Cơ chế Load Balancing của GLBP (AVG/AVF)

GLBP tách vai trò thành 2 cấp:

- **AVG (Active Virtual Gateway)**: 1 router duy nhất được bầu (dựa trên Priority, giống Active của HSRP) — chịu trách nhiệm trả lời ARP request cho VIP, nhưng **không nhất thiết** là router duy nhất xử lý traffic.
- **AVF (Active Virtual Forwarder)**: AVG gán **các Virtual MAC khác nhau** cho từng router trong nhóm (bao gồm cả chính nó), biến mỗi router thành 1 AVF. Khi PC ARP hỏi VIP, AVG trả lời **luân phiên** các VMAC khác nhau cho các PC khác nhau — kết quả là các PC khác nhau trong cùng mạng thực chất đang gửi traffic ra qua **các router vật lý khác nhau**, dù tất cả đều nghĩ mình dùng chung 1 Gateway.

> [!question] Nếu 1 AVF trong nhóm GLBP gặp sự cố, các PC đang dùng VMAC của AVF đó có mất kết nối không?
> Không. Khi AVF ngừng hoạt động (Weighting giảm dưới ngưỡng cấu hình), AVG phát hiện và **thu hồi lại VMAC đó**, chuyển giao trách nhiệm forward sang 1 AVF khác còn hoạt động — các PC vẫn tiếp tục dùng đúng VMAC cũ đã ARP được trước đó, chỉ khác là giờ traffic được router khác xử lý. Đây là lý do GLBP vừa cân bằng tải vừa vẫn giữ được tính dự phòng như HSRP/VRRP.

## 5. Triển khai và cấu hình HSRP

```
R1(config)# interface gigabitEthernet 0/1
R1(config-if)# standby 1 ip 10.10.10.1
R1(config-if)# standby 1 priority 150
R1(config-if)# standby 1 preempt
```

```
R2(config)# interface gigabitEthernet 0/1
R2(config-if)# standby 1 ip 10.10.10.1
R2(config-if)# standby 1 priority 100
R2(config-if)# standby 1 preempt
```

Số `1` sau `standby` là **group number** — phải giống nhau trên cả 2 router để chúng nhận ra nhau thuộc cùng 1 nhóm HSRP. R1 có Priority 150 > R2 Priority 100 (mặc định) nên R1 trở thành Active.

## 6. Triển khai và cấu hình GLBP

```
R1(config)# interface gigabitEthernet 0/1
R1(config-if)# glbp 1 ip 10.10.10.1
R1(config-if)# glbp 1 priority 150
```

```
R2(config)# interface gigabitEthernet 0/1
R2(config-if)# glbp 1 ip 10.10.10.1
R2(config-if)# glbp 1 priority 100
```

Cú pháp tương tự HSRP nhưng chỉ cần **1 GLBP group duy nhất** trên cả 2 router — khác với cách làm "load sharing" thủ công kiểu cũ của HSRP (phải tạo 2 group HSRP riêng, mỗi group 1 router làm Active, rồi chia tay 1 nửa số PC dùng Gateway group này, nửa còn lại dùng Gateway group kia) — GLBP tự động hóa hoàn toàn việc chia tải mà không cần cấu hình phức tạp đó.

## 7. Object Tracking — theo dõi cả đường lên WAN, không chỉ interface LAN

Một lỗi thiết kế phổ biến: Active Router chỉ theo dõi trạng thái interface **hướng vào LAN** — nếu đường **WAN/Uplink** của chính Active Router bị đứt (nhưng interface LAN vẫn up), FHRP không hề biết gì bất thường và **vẫn giữ nguyên vai trò Active**, khiến PC tiếp tục gửi traffic qua đúng router đã mất đường ra ngoài — gây mất kết nối dù về lý thuyết "vẫn có Active Router hoạt động".

```
R1(config)# track 1 interface gigabitEthernet 0/0 line-protocol
R1(config)# interface gigabitEthernet 0/1
R1(config-if)# standby 1 track 1 decrement 60
```

`track 1 interface gigabitEthernet 0/0 line-protocol` theo dõi trạng thái đường uplink (`Gi0/0`). Nếu uplink down, Priority của R1 tự động giảm `60` điểm — nếu mức giảm đủ lớn để Priority R1 xuống thấp hơn R2, R2 chỉ chủ động giành lại vai trò Active nếu **chính R2 có `standby 1 preempt`** (như cấu hình R2 ở mục 5) — nếu R2 không bật preempt, R2 vẫn ở Standby dù Priority của R1 đã thấp hơn; việc R1 bị trừ Priority bởi `track` không tự khiến R2 tiếp quản. Cấu hình này giải quyết đúng lỗ hổng nêu trên.

## 8. Giám sát hoạt động và xử lý sự cố

```
Router# show standby brief
Router# show standby
Router# show glbp brief
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Cả 2 router cùng báo "Active" | Group number không khớp giữa 2 router, hoặc mất kết nối Layer 2 khiến chúng không nghe được Hello của nhau (split-brain) | `show standby brief` đối chiếu group number 2 đầu; kiểm tra kết nối Layer 2 (VLAN/trunk) giữa 2 router có đang hoạt động bình thường không |
| Router Priority cao hơn phục hồi nhưng không giành lại vai trò Active | Chưa cấu hình `standby preempt` (mặc định tắt trên HSRP, mục 3) | Thêm `standby preempt` nếu muốn hành vi này |
| PC mất kết nối dù "Active Router" vẫn báo hoạt động bình thường | Active Router mất đường WAN/Uplink nhưng FHRP không theo dõi uplink, chỉ theo dõi interface LAN (mục 7) | Cấu hình `track` object theo dõi uplink, kèm `decrement` đủ lớn để kích hoạt chuyển đổi vai trò |
| GLBP chỉ 1 router thực sự forward traffic, không thấy chia tải | AVF còn lại chưa đạt điều kiện tham gia (Weighting dưới ngưỡng), hoặc chỉ có 1 router trong nhóm đang hoạt động | `show glbp brief` xác nhận có đủ AVF đang active; kiểm tra Weighting từng router |

## 9. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 2 router (hoặc L3 switch), 1 PC.

1. Cấu hình HSRP group 1 trên 2 router theo mục 5, xác nhận PC ping thông VIP và `show standby brief` hiển thị đúng 1 Active, 1 Standby.
2. `shutdown` interface LAN của Active Router, quan sát Standby tự động tiếp quản, PC vẫn ping thông VIP không gián đoạn dài (chỉ chờ đúng Hold time).
3. Bật lại interface, quan sát router cũ có giành lại Active hay không tùy có/không có `preempt` — đối chiếu mục 3.
4. Cấu hình thêm `track` cho interface uplink giả lập theo mục 7, `shutdown` đúng interface uplink đó (không phải interface LAN), xác nhận vai trò Active vẫn chuyển giao đúng dù interface LAN chưa hề down (R2 phải có `standby 1 preempt` như mục 5 thì mới giành lại Active khi Priority R1 bị `track` trừ xuống thấp hơn R2).

**Kết quả kỳ vọng**: hiểu và tự tay chứng minh được cơ chế chuyển đổi Active/Standby của HSRP, cũng như lỗ hổng thực chiến phổ biến khi không theo dõi đường uplink.

## 10. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| PC cần cấu hình lại Gateway khi router Active đổi sang router khác | Bỏ qua khái niệm Virtual IP/MAC | PC luôn dùng đúng 1 VIP/VMAC cố định, không cần biết hay thay đổi gì khi vai trò Active chuyển đổi phía sau |
| HSRP và GLBP cho hiệu năng tương đương, chỉ khác cách cấu hình | Bỏ qua khác biệt cốt lõi dự phòng thuần túy vs cân bằng tải thực sự | GLBP cho phép nhiều router cùng active forward traffic thật sự cùng lúc, HSRP/VRRP chỉ có đúng 1 router xử lý tại 1 thời điểm (mục 1) |
| Preempt luôn nên bật vì "Priority cao hơn nên làm Active" | Bỏ qua rủi ro router vừa phục hồi chưa ổn định hoàn toàn | Cần cân nhắc: bật preempt giúp đúng thiết kế Priority nhưng có thể gây chuyển đổi vai trò 2 lần liên tiếp nếu router phục hồi chưa thực sự ổn định |
| FHRP tự động phát hiện mọi loại sự cố ảnh hưởng tới khả năng định tuyến của Active Router | Bỏ qua việc FHRP mặc định chỉ theo dõi trạng thái Hello giữa các router, không tự biết đường WAN có đứt hay không | Cần cấu hình `track` object tường minh để FHRP phản ứng đúng với sự cố uplink (mục 7) |

## 11. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. FHRP giải quyết vấn đề gì mà STP/EtherChannel (đã học ở Bài 10) không giải quyết được?**
→ STP/EtherChannel giải quyết dự phòng ở Lớp 2 (đường truyền vật lý). FHRP giải quyết dự phòng ở Lớp 3 cho chính địa chỉ Gateway mà PC dùng — PC chỉ khai báo được 1 Gateway duy nhất, FHRP cho phép nhiều router giả lập chung 1 Gateway ảo để PC không cần biết/thay đổi gì khi router vật lý phía sau gặp sự cố.

**2. Vì sao GLBP được xem là "cân bằng tải" còn HSRP/VRRP chỉ là "dự phòng"?**
→ Vì HSRP/VRRP tại 1 thời điểm chỉ có đúng 1 router (Active/Master) xử lý traffic, router còn lại chờ sẵn không dùng tới. GLBP dùng cơ chế AVG phân phối nhiều Virtual MAC cho nhiều AVF khác nhau, khiến nhiều router cùng thực sự forward traffic đồng thời.

**3. Preempt là gì? Vì sao mặc định tắt trên HSRP?**
→ Preempt cho phép router Priority cao hơn giành lại vai trò Active ngay khi phục hồi. Mặc định tắt trên HSRP để tránh gián đoạn traffic thêm 1 lần nữa ngay sau khi router vừa phục hồi (có thể chưa hoàn toàn ổn định) lập tức giành lại vai trò.

**4. Object Tracking trong FHRP dùng để giải quyết vấn đề gì?**
→ Giải quyết lỗ hổng: mặc định FHRP chỉ theo dõi trạng thái Hello giữa các router qua interface LAN, không biết đường WAN/Uplink của Active Router có đứt hay không. Track object theo dõi thêm interface uplink, tự động giảm Priority khi uplink down để router khác có thể giành lại vai trò Active.

**5. Virtual MAC của HSRP group 5 sẽ có dạng như thế nào?**
→ `0000.0c07.ac05` — theo cấu trúc cố định `0000.0c07.ac<XX>`, với `XX` là số group HSRP viết dưới dạng hex (group 5 = `05`).

## 12. Tài liệu tham khảo

- [NetworkLessons — Hot Standby Router Protocol (HSRP)](https://networklessons.com/cisco/ccie-routing-switching/hsrp-hot-standby-routing-protocol)
- [PingLabz — GLBP: Cisco's Load-Balancing First Hop Redundancy](https://www.pinglabz.com/glbp-load-balancing/)
- [NetworkAcademy.IO — Virtual Router Redundancy Protocol (VRRP)](https://www.networkacademy.io/ccna/network-services/virtual-router-redundancy-protocol-vrrp)
- [Cisco — Troubleshoot HSRP Common Issues](https://www.cisco.com/c/en/us/support/docs/ip/hot-standby-router-protocol-hsrp/10583-62.html)

---
**Liên kết**: [[Bai-20_NAT|← Bài 20]] · [[Bai-22_Remote-Access-VPN|Bài 22 - Remote Access – VPN →]]
