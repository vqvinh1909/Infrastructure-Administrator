---
title: Bài 12 - Inter-VLAN Routing
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — Router-on-a-stick và SVI là 2 cách triển khai Inter-VLAN Routing bắt buộc phải phân biệt được cả về cấu hình lẫn hiệu năng thực tế"
tags: [networking, cisco, routing, inter-vlan, router-on-a-stick, svi, sysadmin]
aliases: [Inter-VLAN Routing, Router-on-a-Stick, SVI, Switch Virtual Interface]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-08_Cau-hinh-va-quan-ly-VLAN]]", "[[Bai-09_Trunking-DTP-VTP]]"]
last_review: 2026-07-21
---

# Bài 12: Inter-VLAN Routing

> [!important] Tại sao cần file này
> [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] đã xác nhận: 2 PC khác VLAN cần 1 Router đứng giữa mới thông nhau. Nhưng nếu doanh nghiệp có 5, 10 VLAN, chẳng lẽ phải kéo 5-10 sợi cáp riêng từ switch tới router, mỗi sợi 1 VLAN? Bài này giới thiệu 2 cách giải quyết vấn đề đó trong thực tế: **Router-on-a-Stick** (dùng 1 sợi cáp trunk duy nhất) và **SVI trên Layer 3 Switch** (định tuyến ngay trên switch, không cần router rời) — đây là 2 kỹ thuật bắt buộc phải nắm khi triển khai mạng doanh nghiệp có nhiều VLAN.

## 1. Router-on-a-Stick

Thay vì kéo nhiều cáp (mỗi cáp 1 VLAN, đúng theo mô tả ở [[Bai-09_Trunking-DTP-VTP|Bài 9]] mục 1), Router-on-a-Stick dùng **1 sợi cáp trunk duy nhất** giữa switch và router, sau đó chia interface vật lý của router thành nhiều **subinterface** logic — mỗi subinterface đại diện cho 1 VLAN.

```
+------------+     +--------------------+     +------------+
|    PC-A    |-----|       SWITCH       |-----|    PC-B    |
|  VLAN 10   |     |   (access+trunk)   |     |  VLAN 20   |
+------------+     +--------------------+     +------------+

                              |
                        trunk (802.1Q)
                              |
                  +----------------------+
                  |        ROUTER        |
                  |  Gi0/0.10 (VLAN 10)  |
                  |  Gi0/0.20 (VLAN 20)  |
                  +----------------------+
```

**Cấu hình:**

```
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# no shutdown
Router(config-if)# exit
Router(config)# interface gigabitEthernet 0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 10.10.10.1 255.255.255.0
Router(config-subif)# exit
Router(config)# interface gigabitEthernet 0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 10.10.20.1 255.255.255.0
Router(config-subif)# exit
```

Phía switch, cổng nối tới router phải cấu hình trunk (đúng như đã học ở [[Bai-09_Trunking-DTP-VTP|Bài 9]] mục 5):

```
Switch(config)# interface gigabitEthernet 0/24
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20
```

> [!question] Vì sao chỉ cần `no shutdown` trên interface vật lý (`Gi0/0`) mà không cần `ip address` trên chính nó?
> Vì địa chỉ IP được gán ở cấp **subinterface** (`Gi0/0.10`, `Gi0/0.20`), không phải interface vật lý gốc. Interface vật lý chỉ đóng vai trò "đường ống" mang traffic đã gắn tag 802.1Q — bản thân nó không cần và thường không nên có địa chỉ IP riêng, chỉ cần ở trạng thái up (`no shutdown`) để các subinterface phía trên hoạt động được.

## 2. SVI trên Layer 3 Switch

**SVI (Switch Virtual Interface)** là 1 interface ảo đại diện cho 1 VLAN, được tạo ngay trên switch hỗ trợ Layer 3 (Layer 3 Switch / Multilayer Switch) — router-on-a-stick không còn cần thiết vì việc định tuyến diễn ra **ngay trên switch**, ở tốc độ phần cứng (hardware, wire-speed) thay vì phải "đẩy" gói tin ra ngoài cho 1 router rời xử lý bằng phần mềm.

```
Layer 3 Switch voi SVI (dinh tuyen ngay tren switch):
+------------+    +--------------+    +------------+
|    PC-A    |    |  L3 SWITCH   |    |    PC-B    |
|  VLAN 10   |----|  SVI VLAN10  |----|  VLAN 20   |
|            |    |  SVI VLAN20  |    |            |
+------------+    +--------------+    +------------+
```

**Cấu hình:**

```
Switch(config)# ip routing
Switch(config)# vlan 10
Switch(config-vlan)# exit
Switch(config)# interface vlan 10
Switch(config-if)# ip address 10.10.10.1 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# exit
Switch(config)# interface vlan 20
Switch(config-if)# ip address 10.10.20.1 255.255.255.0
Switch(config-if)# no shutdown
```

> [!question] Vì sao lệnh `ip routing` bắt buộc phải có, dù đã tạo đủ SVI?
> Vì mặc định switch (kể cả Layer 3 Switch) khởi động ở chế độ **chỉ chuyển mạch Lớp 2**, để tránh vô tình biến switch thành router khi chưa sẵn sàng. `ip routing` là lệnh "bật công tắc" cho phép switch thực sự định tuyến giữa các SVI — thiếu lệnh này, các SVI vẫn nhận IP và ping được chính switch, nhưng traffic **không được định tuyến** giữa các VLAN với nhau.

## 3. So sánh Router-on-a-Stick và SVI

| Tiêu chí | Router-on-a-Stick | SVI (Layer 3 Switch) |
|---|---|---|
| Thiết bị cần | Router rời + Switch | Chỉ cần Layer 3 Switch |
| Nơi xử lý định tuyến | Phần mềm (CPU router) | Phần cứng (ASIC switch) — nhanh hơn nhiều |
| Băng thông định tuyến | Giới hạn bởi băng thông cổng trunk vật lý duy nhất — mọi VLAN chia sẻ chung 1 cổng | Không giới hạn bởi 1 cổng — định tuyến ngay trong nội bộ switch ở tốc độ line-rate |
| Độ trễ (latency/jitter) | Cao hơn — gói tin phải đi ra router rồi quay lại switch | Thấp hơn đáng kể vì không cần "đi vòng" |
| Khả năng mở rộng | Không khuyến nghị khi số VLAN lớn (thực tế thường giới hạn hiệu quả dưới 50 VLAN) | Mở rộng tốt hơn nhiều, phù hợp mạng doanh nghiệp lớn |
| Chi phí | Tận dụng được router cũ sẵn có | Cần switch hỗ trợ Layer 3 (thường đắt hơn switch Layer 2 thuần) |
| Trường hợp dùng phổ biến | Mạng nhỏ, phòng lab, hoặc nơi đã có sẵn router chưa cần nâng cấp | Hầu hết mạng doanh nghiệp sản xuất hiện nay, đặc biệt ở lớp Distribution |

> [!question] Vì sao gần như mọi thiết kế mạng doanh nghiệp hiện đại đều ưu tiên SVI hơn Router-on-a-Stick?
> Vì Router-on-a-Stick có 1 điểm nghẽn (bottleneck) cố định: **toàn bộ traffic liên VLAN đều phải chia sẻ chung 1 cổng trunk vật lý** giữa switch và router — dù cổng đó là Gigabit hay 10 Gigabit, nó vẫn là giới hạn cứng. SVI loại bỏ hoàn toàn điểm nghẽn này vì việc định tuyến diễn ra ngay trong switch, không cần "đẩy" traffic ra một thiết bị khác rồi nhận lại. Đây là lý do kiến trúc Two-Tier/Three-Tier đã học ở [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]] (mục 5) luôn đặt định tuyến liên VLAN ở lớp Distribution bằng Layer 3 Switch, không dùng router rời.

## 4. Verify & Troubleshoot

| Lệnh | Công dụng |
|---|---|
| `show ip route` | Xác nhận các subnet VLAN đã xuất hiện dạng `C`/`L` trong routing table |
| `show interfaces gigabitEthernet 0/0.10` | Xem trạng thái, encapsulation của 1 subinterface cụ thể (Router-on-a-Stick) |
| `show vlan brief` | Xác nhận VLAN đã tồn tại trước khi tạo SVI tương ứng (SVI) |
| `show interfaces vlan 10` | Xem trạng thái up/down của 1 SVI cụ thể |
| `show ip interface brief` | Xem nhanh toàn bộ interface (vật lý + subinterface/SVI) cùng trạng thái IP |

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| 2 PC khác VLAN vẫn không ping thông dù đã cấu hình subinterface/SVI đầy đủ | SVI/subinterface đúng nhưng thiếu `ip routing` (SVI) hoặc cổng phía switch chưa đúng trunk allowed VLAN | Kiểm tra `ip routing` đã bật (SVI); đối chiếu `show interfaces trunk` xem VLAN đã được allowed qua trunk chưa |
| SVI luôn ở trạng thái down dù VLAN đã tồn tại | SVI mặc định "down" nếu VLAN đó không có ít nhất 1 cổng L2 (access hoặc trunk cho phép VLAN đó) ở trạng thái link up và STP forwarding trong VLAN đó, hoặc VLAN chưa tồn tại/active (hành vi thiết kế, không phải lỗi) | Xác nhận VLAN tồn tại/active và có ít nhất 1 cổng L2 (access hoặc trunk) link up, STP forwarding trong VLAN; hoặc dùng `no autostate` dưới `interface vlan X` (tùy nền tảng/IOS) hay `switchport autostate exclude` trên cổng L2 để loại cổng đó khỏi tính toán autostate |
| Subinterface router không nhận traffic dù đã đúng VLAN ID | Sai `encapsulation dot1Q <id>` không khớp VLAN thực tế, hoặc quên `no shutdown` interface vật lý gốc | `show interfaces gigabitEthernet 0/0.10` đối chiếu đúng VLAN ID; kiểm tra interface vật lý gốc đã up/up |
| Traffic liên VLAN chậm bất thường dù mạng không lỗi | Đang dùng Router-on-a-Stick với nhiều VLAN, cổng trunk trở thành điểm nghẽn (mục 3) | Cân nhắc chuyển sang SVI trên Layer 3 Switch nếu số VLAN/traffic liên VLAN đã lớn |

## 5. Lab thực hành

**Môi trường**: Cisco Packet Tracer.

1. Bài lab A — Router-on-a-Stick: 1 router, 1 switch, 2 PC ở VLAN 10 và 20. Cấu hình trunk trên switch, subinterface trên router theo mục 1. Xác nhận 2 PC ping thông nhau qua router.
2. Bài lab B — SVI: dùng 1 Layer 3 Switch (loại `Multilayer Switch` trong Packet Tracer), lặp lại với 2 PC tương tự nhưng không dùng router rời — chỉ cấu hình SVI theo mục 2. Xác nhận 2 PC vẫn ping thông nhau.
3. So sánh: xóa cấu hình router ở lab A (`shutdown` interface trunk), quan sát 2 PC ở lab A mất kết nối ngay — trong khi lab B không phụ thuộc thiết bị rời nào khác ngoài chính switch.

**Kết quả kỳ vọng**: tự tay cấu hình và phân biệt được 2 phương pháp Inter-VLAN Routing, hiểu rõ vì sao SVI là lựa chọn ưu tiên trong hạ tầng doanh nghiệp thực tế.

## 6. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Router-on-a-Stick và SVI cho hiệu năng tương đương, chỉ khác cách cấu hình | Bỏ qua khác biệt xử lý phần mềm (router) và phần cứng (switch ASIC) | SVI nhanh hơn đáng kể vì định tuyến ở tốc độ line-rate ngay trên switch, Router-on-a-Stick bị giới hạn bởi băng thông 1 cổng trunk |
| Chỉ cần tạo SVI với đúng IP là switch tự định tuyến được | Quên rằng switch mặc định chỉ hoạt động Lớp 2 | Bắt buộc phải có `ip routing` thì switch mới thực sự định tuyến giữa các SVI |
| Interface vật lý của router trong Router-on-a-Stick cũng cần đặt IP riêng | Nhầm với cấu hình interface thông thường không dùng subinterface | IP chỉ đặt ở từng subinterface; interface vật lý gốc chỉ cần `no shutdown` |
| SVI luôn up ngay khi tạo, không phụ thuộc gì khác | Bỏ qua cơ chế autostate của SVI | SVI mặc định phụ thuộc có ít nhất 1 cổng L2 (access hoặc trunk cho phép VLAN đó) đang link up và STP forwarding trong VLAN, trừ khi cấu hình `no autostate` (dưới `interface vlan X`, tùy nền tảng) hoặc `switchport autostate exclude` (trên cổng L2) |

## 7. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Router-on-a-Stick giải quyết vấn đề gì so với việc nối nhiều cáp riêng cho từng VLAN?**
→ Thay vì kéo 1 cáp riêng cho mỗi VLAN, Router-on-a-Stick dùng 1 cáp trunk 802.1Q duy nhất mang tất cả VLAN, sau đó chia interface router thành nhiều subinterface — mỗi subinterface xử lý đúng 1 VLAN thông qua `encapsulation dot1Q`.

**2. SVI là gì, và vì sao cần lệnh `ip routing` để nó hoạt động đúng?**
→ SVI là interface ảo đại diện cho 1 VLAN trên Layer 3 Switch, cho phép định tuyến ngay trên switch. Mặc định switch chỉ hoạt động Lớp 2 để tránh vô tình bật định tuyến ngoài ý muốn; `ip routing` là lệnh tường minh bật khả năng định tuyến giữa các SVI.

**3. Vì sao SVI thường được ưu tiên hơn Router-on-a-Stick trong mạng doanh nghiệp sản xuất?**
→ Vì Router-on-a-Stick có điểm nghẽn cố định là băng thông của 1 cổng trunk vật lý duy nhất mà mọi VLAN phải chia sẻ, còn SVI định tuyến ngay trong phần cứng switch (ASIC) ở tốc độ line-rate, không bị giới hạn bởi 1 cổng vật lý, phù hợp hơn khi số VLAN và traffic liên VLAN lớn.

**4. Vì sao interface vật lý gốc trong Router-on-a-Stick không cần địa chỉ IP riêng?**
→ Vì IP được gán ở cấp subinterface tương ứng từng VLAN; interface vật lý chỉ đóng vai trò đường truyền mang traffic đã gắn tag 802.1Q, không tham gia định tuyến trực tiếp.

**5. SVI của 1 VLAN bị "down" dù đã cấu hình đúng IP — nguyên nhân phổ biến nhất là gì?**
→ Theo cơ chế autostate mặc định, SVI chỉ up khi VLAN tồn tại/active và có ít nhất 1 cổng L2 (access hoặc trunk cho phép VLAN đó) ở trạng thái link up và STP forwarding trong VLAN — nếu không có cổng L2 nào như vậy đang hoạt động, SVI sẽ tự động down dù cấu hình IP hoàn toàn đúng.

## 8. Tài liệu tham khảo

- [ComputingForGeeks — Configure Layer 3 Switch SVI Routing](https://computingforgeeks.com/cisco-layer3-switch-inter-vlan-svi/)
- [Cisco Press — Inter-VLAN Routing using Layer 3 Switches](https://www.ciscopress.com/articles/article.asp?p=3089357&seqNum=6)
- [PingLabz — Inter-VLAN Routing: SVI vs Router-on-a-Stick on Cisco IOS XE](https://www.pinglabz.com/inter-vlan-routing/)

---
**Liên kết**: [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|← Bài 11]] · [[Bai-13_Static-Route|Bài 13 - Static Route →]]
