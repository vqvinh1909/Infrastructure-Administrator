---
title: Bài 14 - OSPF
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — OSPF là giao thức Link State phổ biến nhất trong doanh nghiệp, gần như luôn xuất hiện trong bài test thực hành và câu hỏi phỏng vấn Mid-level"
tags: [networking, cisco, routing, ospf, link-state, dr-bdr, sysadmin]
aliases: [OSPF, Open Shortest Path First, Link State, DR, BDR]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-13_Static-Route]]", "[[Bai-15_EIGRP]]"]
last_review: 2026-07-21
---

# Bài 14: OSPF (Open Shortest Path First)

> [!important] Tại sao cần file này
> [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] (mục 6) đã phân loại OSPF là giao thức **Link State** — khác hẳn cách Static Route ([[Bai-13_Static-Route|Bài 13]]) yêu cầu khai báo tay từng đường đi. OSPF là giao thức định tuyến động **mở (không độc quyền hãng nào)** phổ biến nhất trong hạ tầng doanh nghiệp thực tế, cho phép router tự động học và cập nhật đường đi khi topology thay đổi mà không cần con người can thiệp lại từng route.

## 1. Cơ chế Link State — vì sao khác Distance Vector

Router chạy OSPF không chỉ biết "hướng đi và khoảng cách" như Distance Vector — mỗi router tự xây dựng 1 **bản đồ đầy đủ (LSDB — Link State Database)** về toàn bộ topology trong phạm vi của nó, bằng cách trao đổi thông tin LSA (Link State Advertisement) mô tả từng liên kết (link) nó thấy được. Sau khi có LSDB đầy đủ và **giống hệt nhau** trên mọi router trong cùng vùng, mỗi router tự chạy thuật toán **Dijkstra (SPF — Shortest Path First)** để tính ra đường ngắn nhất tới từng đích, độc lập với router khác.

## 2. Router ID

Mỗi router OSPF cần 1 **Router ID (RID)** duy nhất trong toàn vùng — dùng để nhận diện router đó trong LSDB. Thứ tự ưu tiên chọn RID nếu không cấu hình tay:

1. RID cấu hình tường minh bằng lệnh `router-id` (luôn được ưu tiên cao nhất nếu có).
2. IP cao nhất trong các interface **Loopback** đang up (loopback không phụ thuộc trạng thái vật lý nên thường được khuyến nghị dùng làm RID ổn định).
3. IP cao nhất trong các interface vật lý đang up nếu không có Loopback nào.

> [!question] Vì sao 2 router trùng Router ID lại gây ra sự cố khó phát hiện?
> Vì OSPF dùng Router ID để phân biệt "ai là ai" trong LSDB — nếu 2 router (đặc biệt phổ biến khi dùng chung IP loopback mặc định hoặc nhân bản cấu hình từ 1 template) vô tình trùng RID, các router còn lại trong vùng có thể nhận LSA từ 2 nguồn khác nhau nhưng tưởng là cùng 1 router, dẫn đến LSDB không đồng bộ đúng, gây route "nhấp nháy" liên tục hoặc route sai mà không có thông báo lỗi rõ ràng — đây là nguyên nhân của nhiều sự cố OSPF thực tế được ghi nhận trong tài liệu troubleshooting của các hãng thiết bị.

## 3. Area — Area 0 (Backbone)

OSPF chia mạng thành các **Area** (vùng) để giới hạn phạm vi LSDB, giảm tải tính toán SPF khi mạng lớn. Mọi thiết kế OSPF **bắt buộc phải có Area 0 (Backbone Area)**, và mọi area khác phải kết nối trực tiếp (hoặc qua liên kết ảo) tới Area 0. Trong phạm vi bài này, ta tập trung vào **Single-Area OSPF** (toàn bộ router chung 1 Area 0) — cách triển khai đơn giản và phổ biến nhất cho mạng doanh nghiệp vừa/nhỏ; thiết kế Multi-Area (nhiều Area, ABR/ASBR) là chủ đề nâng cao hơn, nằm ngoài phạm vi Sysadmin Junior/Mid-level.

## 4. Điều kiện thiết lập Neighbor

2 router OSPF chỉ hình thành được quan hệ neighbor khi **toàn bộ** các thông số sau khớp nhau giữa 2 đầu:

| Điều kiện | Chi tiết |
|---|---|
| **Area ID** | Interface 2 đầu phải khai báo cùng 1 Area (mục 3) |
| **Subnet** | Trên mạng Ethernet (Broadcast), 2 interface phải cùng subnet; ở liên kết Serial point-to-point, subnet có thể khác nhau tùy thiết kế |
| **Hello Interval / Dead Interval** | Phải khớp tuyệt đối 2 đầu. Mặc định trên Ethernet (Broadcast): **Hello = 10 giây, Dead = 40 giây** (Dead luôn gấp 4 lần Hello theo mặc định); trên liên kết NBMA: Hello = 30 giây, Dead = 120 giây |
| **Router ID** | Phải là duy nhất — 2 router trùng RID sẽ gây sự cố (mục 2), không hình thành neighbor đúng cách |
| **Authentication** | Nếu 1 đầu bật xác thực (authentication), đầu kia phải cấu hình khớp (cùng phương thức, cùng mật khẩu/khóa) |
| **MTU** | Nên khớp nhau; nếu không khớp, neighbor vẫn có thể lên nhưng **không trao đổi được** Database Description đầy đủ — cần `ip ospf mtu-ignore` nếu buộc phải chấp nhận sai khác |
| **Stub Area flag** | Cờ đánh dấu loại area (stub/totally stub/NSSA) trong Hello packet phải khớp nhau (nằm ngoài phạm vi Single-Area của bài này, chỉ cần biết là 1 điều kiện tồn tại) |

> [!question] Vì sao Hello/Dead Interval "phải khớp tuyệt đối" trong khi Cisco cho phép chỉnh tay 2 thông số này?
> Vì Hello/Dead Interval quyết định "nhịp" trao đổi tín hiệu sống giữa 2 router — nếu 2 đầu chỉnh timer khác nhau (ví dụ 1 bên Hello 5 giây, bên kia vẫn mặc định 10 giây), router sẽ liên tục hiểu sai trạng thái của nhau (một bên nghĩ hàng xóm còn sống dựa trên nhịp riêng của mình, bên kia lại tính theo nhịp khác), khiến OSPF **từ chối hình thành neighbor** ngay từ bước Hello đầu tiên, dù mọi thông số khác đều đúng. Đây là lý do khi chỉnh Hello/Dead Interter tay, luôn phải áp dụng đồng bộ trên toàn bộ router trong cùng segment.

## 5. Cơ chế trao đổi LSDB: Point-to-Point và Broadcast Multi-Access

OSPF phân loại mạng theo kiểu kết nối, ảnh hưởng trực tiếp tới cách 2 router trao đổi LSDB:

- **Point-to-Point** (ví dụ liên kết Serial giữa đúng 2 router): chỉ có đúng 2 thiết bị trên liên kết, nên **không cần bầu DR/BDR** — 2 router luôn hình thành Full-adjacency trực tiếp với nhau ngay khi Hello khớp.
- **Broadcast Multi-Access** (ví dụ Ethernet, nơi nhiều router cùng chung 1 segment): nếu để mọi router hình thành Full-adjacency với nhau, số lượng adjacency sẽ tăng theo bình phương số router (full-mesh là n(n-1)/2) không cần thiết — OSPF giải quyết bằng cách bầu ra **DR (Designated Router)** và **BDR (Backup Designated Router)**, mọi router khác chỉ cần hình thành Full-adjacency với DR và BDR, không cần với nhau.

> [!question] Vì sao Point-to-Point không cần DR/BDR nhưng Broadcast Multi-Access thì cần?
> Vì DR/BDR tồn tại để giảm số lượng adjacency trên 1 segment có **nhiều hơn 2 router**. Point-to-Point theo định nghĩa chỉ có đúng 2 thiết bị, nên không có sự bùng nổ số adjacency nào để tối ưu — 2 router cứ trực tiếp Full-adjacency với nhau là đủ, thêm cơ chế DR/BDR vào trường hợp này chỉ gây phức tạp không cần thiết.

**Bầu chọn DR/BDR trên mạng Broadcast Multi-Access:**

```
+----------+  +----------+  +----------+
|    R1    |  |    R2    |  |    R3    |
|    DR    |  |   BDR    |  | DROTHER  |
+----------+  +----------+  +----------+
========================================
Ethernet Segment (multi-access) - DR/BDR chi bau tren mang nay
```

**Tiêu chí bầu chọn** (theo thứ tự):

1. Router có **OSPF Priority cao nhất** thắng (mặc định Priority = 1 trên mọi interface; đặt `0` nghĩa là router đó **không bao giờ** tham gia làm DR/BDR).
2. Nếu Priority bằng nhau, router có **Router ID cao nhất** thắng.

> [!question] Vì sao DR/BDR không "chuyển giao lại" khi có router Priority cao hơn tham gia sau?
> Vì cơ chế bầu chọn OSPF DR/BDR là **non-preemptive** (không tranh giành lại) — một khi DR/BDR đã được bầu ổn định, dù sau đó có router mới tham gia mạng với Priority cao hơn, DR/BDR hiện tại **vẫn giữ nguyên vai trò** cho đến khi chính nó gặp sự cố (down). Thiết kế này tránh việc DR/BDR bị đổi liên tục gây gián đoạn không cần thiết mỗi khi có thiết bị mới join mạng.

## 6. Cost — Metric của OSPF

OSPF dùng **Cost** làm metric, tính theo công thức:

```
Cost = Reference Bandwidth / Bandwidth của interface
```

Reference Bandwidth mặc định là **100 Mbps** trên Cisco IOS — nghĩa là 1 interface Gigabit Ethernet (1000 Mbps) sẽ có cost mặc định = 100/1000 = 1 (làm tròn), trong khi 1 interface FastEthernet (100 Mbps) có cost = 100/100 = 1 cũng — hai loại interface có cost bằng nhau dù tốc độ chênh lệch 10 lần, vì Reference Bandwidth mặc định đã lỗi thời so với tốc độ mạng hiện đại.

> [!question] Vì sao Reference Bandwidth mặc định 100 Mbps lại là 1 vấn đề trong mạng hiện đại?
> Vì công thức Cost bị "bão hòa" ở giá trị 1 với mọi interface từ Gigabit trở lên — router không còn phân biệt được Gigabit với 10 Gigabit hay 40 Gigabit về mặt cost, dẫn đến việc chọn đường đi tối ưu có thể sai lệch so với thực tế. Cách khắc phục: cấu hình lại `auto-cost reference-bandwidth <giá trị mới>` cao hơn (ví dụ 100000 cho 100 Gbps) trên **toàn bộ router trong vùng** — phải đồng nhất, nếu không tính toán SPF sẽ không khớp nhau.

## 7. Các trạng thái Neighbor (Neighbor States)

```
Down -> Init -> 2-Way -> ExStart -> Exchange -> Loading -> Full
```

| Trạng thái | Ý nghĩa |
|---|---|
| **Down** | Chưa nhận được Hello packet nào từ hàng xóm |
| **Init** | Đã nhận Hello nhưng chưa thấy Router ID của chính mình trong đó (chưa xác nhận 2 chiều) |
| **2-Way** | Cả 2 bên đã thấy Router ID của nhau trong Hello — với router **không phải** DR/BDR, quan hệ có thể dừng ở đây (không cần Full) |
| **ExStart, Exchange, Loading** | Đang trao đổi và đồng bộ LSDB (Database Description, Link State Request/Update) |
| **Full** | LSDB đã đồng bộ hoàn toàn — trạng thái ổn định, bình thường của 1 adjacency hoàn chỉnh |

Trên mạng multi-access, router thường chỉ đạt **Full** với DR và BDR, còn giữ **2-Way** với các router `DROTHER` khác — đây là hành vi thiết kế đúng, không phải lỗi.

## 8. Cấu hình OSPFv2 Single-Area (IPv4)

```
Router(config)# router ospf 1
Router(config-router)# router-id 1.1.1.1
Router(config-router)# network 10.10.10.0 0.0.0.255 area 0
Router(config-router)# network 10.10.20.0 0.0.0.255 area 0
```

- `router ospf 1`: số `1` là **Process ID**, chỉ có ý nghĩa cục bộ trên router đó, không cần khớp giữa các router khác nhau.
- `network <địa chỉ> <wildcard mask> area <id>`: khai báo interface nào tham gia OSPF, dựa trên **wildcard mask** (đã học ở [[Bai-05_Dia-chi-IPv4|Bài 5]]) chứ không phải subnet mask thông thường.

## 9. Cấu hình OSPFv3 (IPv6)

OSPFv3 là phiên bản OSPF dành cho **IPv6** (đã học ở [[Bai-06_Dia-chi-IPv6|Bài 6]]) — khác biệt lớn nhất so với OSPFv2 là interface được kích hoạt **trực tiếp ở chế độ interface configuration**, không dùng câu lệnh `network` như OSPFv2:

```
Router(config)# ipv6 unicast-routing
Router(config)# ipv6 router ospf 1
Router(config-rtr)# router-id 1.1.1.1
Router(config-rtr)# exit
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ipv6 ospf 1 area 0
```

> [!question] Vì sao OSPFv3 không dùng lệnh `network` như OSPFv2 để khai báo interface tham gia?
> Đây là 1 thay đổi thiết kế có chủ đích: OSPFv3 kích hoạt trực tiếp ngay trong chế độ cấu hình từng interface (`ipv6 ospf <process-id> area <area>`), thay vì phải dùng wildcard mask để "khớp" interface từ chế độ router configuration như OSPFv2. Cách làm này rõ ràng hơn, tránh nhầm lẫn do tính sai wildcard mask — 1 lỗi khá phổ biến với người mới khi cấu hình OSPFv2.

> **Lưu ý cập nhật:** Cú pháp `ipv6 router ospf` + `ipv6 ospf <pid> area <area>` ở trên là cú pháp legacy (chỉ dành cho IPv6). Cisco IOS/IOS XE hiện khuyến nghị dùng `router ospfv3 <pid>` kèm `address-family ipv6 unicast`, và trên interface dùng `ospfv3 <pid> ipv6 area 0`; OSPFv3 còn hỗ trợ cả address-family IPv4 (RFC 5838). Cú pháp cũ vẫn dùng được như phương án legacy.

## 10. Default-route trong OSPF

Để router quảng bá 1 default route (`0.0.0.0/0`, đã học ở [[Bai-13_Static-Route|Bài 13]] mục 4) vào toàn vùng OSPF cho các router khác cùng dùng, cần lệnh `default-information originate`:

```
Router(config)# router ospf 1
Router(config-router)# default-information originate
```

Mặc định, lệnh này chỉ hoạt động nếu router đó **đã có sẵn** 1 default route trong routing table (ví dụ từ static route trỏ ra ISP). Muốn quảng bá default route dù **không có** route đó trong bảng, thêm từ khóa `always`:

```
Router(config-router)# default-information originate always
```

> [!question] Vì sao nên cẩn trọng khi dùng `always` thay vì để mặc định?
> Vì `always` bỏ qua hoàn toàn việc kiểm tra router có thực sự biết đường ra ngoài hay không — nếu dùng `always` trên 1 router **không có** đường ra Internet thực sự (ví dụ do đường chính đứt), router đó vẫn tiếp tục quảng bá default route cho toàn vùng OSPF, khiến các router khác định tuyến traffic **về đúng router không có đường đi thật** — một dạng "quảng cáo sai sự thật" tương tự về hậu quả với case study AS7007 (mục 12) dù ở quy mô nhỏ hơn nhiều. Chỉ nên dùng `always` khi chắc chắn router đó luôn có cách khác để xử lý traffic đó (ví dụ chính nó là router biên thật sự).

## 11. Verify OSPF

```
Router# show ip ospf neighbor
Router# show ip ospf interface
Router# show ip route ospf
Router# show ip protocols
```

`show ip ospf neighbor` là lệnh đầu tiên cần chạy khi nghi ngờ OSPF có vấn đề — nếu hàng xóm không xuất hiện hoặc không đạt trạng thái Full (mục 7), gần như chắc chắn có lỗi cấu hình.

## 12. Case study — Router ID trùng lặp gây vòng lặp định tuyến trong mạng PE-CE

> [!example] Case study kỹ thuật — Duplicate Router ID gây routing loop
> Tài liệu troubleshooting của Huawei ghi nhận 1 trường hợp thực tế: 2 thiết bị CE (Customer Edge) trong cùng 1 vùng OSPF multi-instance vô tình có **2 interface được gán cùng 1 địa chỉ IP** dùng làm Router ID — dẫn đến cả 2 router bị trùng RID. Hậu quả: các router còn lại nhận LSA từ 2 nguồn khác nhau nhưng router ID giống hệt nhau khiến LSDB liên tục bị ghi đè lẫn nhau, route default OSPF **thay đổi liên tục** giữa 2 hướng khác nhau, gây mất gói tin và route không ổn định — về hành vi bên ngoài rất giống 1 dạng vòng lặp định tuyến, dù nguyên nhân gốc chỉ là 1 lỗi cấu hình IP tưởng như nhỏ.
>
> **Bài học thực chiến**: đây là lý do nhiều tổ chức chọn dùng địa chỉ **Loopback riêng, không trùng với bất kỳ subnet nào khác** để làm Router ID cố định (mục 2), thay vì để router tự chọn IP interface vật lý cao nhất — cách làm thủ công dễ dẫn đến trùng lặp khi nhân bản cấu hình giữa nhiều router giống nhau. Nguồn: [Huawei — Case Study: OSPF Routing Loops Caused by Router ID Conflicts](https://support.huawei.com/enterprise/en/doc/EDOC1000177634/e36010e8/case-study-ospf-routing-loops-caused-by-router-id-conflicts).

## 13. Verify & Troubleshoot

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| 2 router không hình thành neighbor (mãi ở trạng thái Down/Init) | Sai Area ID 2 đầu, hoặc Hello/Dead Interval không khớp, hoặc interface chưa được `network` đúng wildcard mask | `show ip ospf interface` đối chiếu Area, Hello/Dead Interval 2 đầu; `show ip ospf neighbor` xác nhận đã thấy hàng xóm chưa |
| Neighbor dừng ở `2-Way`, không lên `Full` | Bình thường nếu router đó không phải DR/BDR trên mạng multi-access (mục 7) — không phải lỗi | Chỉ cần lo lắng nếu router đó **nên** là DR/BDR nhưng vẫn kẹt ở 2-Way |
| Route OSPF liên tục "nhấp nháy" (xuất hiện rồi biến mất) trong routing table | Trùng Router ID giữa 2 router (mục 12), hoặc interface không ổn định (flapping) | `show ip ospf neighbor` đối chiếu Router ID từng router; cân nhắc đổi sang Loopback cố định làm Router ID |
| DR/BDR không như kỳ vọng dù đã đặt Priority cao cho 1 router | Bầu chọn OSPF là non-preemptive (mục 5) — router đó tham gia sau khi DR/BDR đã ổn định | Reset lại quá trình bầu chọn bằng cách restart OSPF process trên toàn bộ router segment đó (chỉ nên làm trong bảo trì đã lên kế hoạch) |
| Đường đi được chọn không phải đường "nhanh nhất" theo cảm nhận thực tế | Cost mặc định bị bão hòa ở giá trị 1 với mọi liên kết từ Gigabit trở lên (mục 6) | Cân nhắc điều chỉnh `auto-cost reference-bandwidth` đồng nhất trên toàn vùng nếu mạng có nhiều loại tốc độ liên kết khác nhau |
| 2 router không hình thành neighbor dù Area/IP đều đúng | Hello/Dead Interval, Authentication, hoặc MTU không khớp 2 đầu (mục 4) | `show ip ospf interface` đối chiếu từng thông số bắt buộc phải khớp theo mục 4 |
| Router đã cấu hình `default-information originate` nhưng default route không xuất hiện ở router khác | Router chưa có sẵn default route thật trong bảng, và chưa dùng từ khóa `always` (mục 10) | Kiểm tra `show ip route` có default route chưa; thêm `always` nếu chắc chắn muốn quảng bá dù không có route thật (cân nhắc rủi ro mục 10) |

## 14. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 3 router nối chung 1 mạng Ethernet multi-access.

1. Cấu hình OSPFv2 Area 0 trên cả 3 router theo mục 8, mỗi router 1 Router ID riêng biệt qua Loopback.
2. Chạy `show ip ospf neighbor` trên từng router, xác định ai là DR, ai là BDR, ai là DROTHER dựa theo Router ID (Priority mặc định bằng nhau).
3. Cố tình đặt trùng Router ID cho 2 trong 3 router (hoặc trùng IP loopback), quan sát hiện tượng route bất ổn định trong `show ip route ospf` — liên hệ case study mục 12.
4. Sửa lại Router ID cho đúng riêng biệt, xác nhận mạng ổn định trở lại.
5. Thêm `default-information originate` trên 1 router có static default route trỏ ra ngoài (theo mục 10), xác nhận 2 router còn lại nhận được default route qua `show ip route ospf`.
6. (Tùy chọn, nếu Packet Tracer hỗ trợ) Lặp lại bước 1-2 với IPv6 theo mục 9 (`ipv6 router ospf`, `ipv6 ospf area`), xác nhận neighbor IPv6 hình thành tương tự IPv4.

**Kết quả kỳ vọng**: hiểu và tự tay tái hiện được nguyên nhân gốc rễ của 1 lỗi OSPF thực chiến phổ biến (trùng Router ID), thành thạo cấu hình OSPFv2/OSPFv3 cơ bản và quảng bá default route vào OSPF.

## 15. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| OSPF Process ID phải giống nhau giữa các router mới hoạt động được | Nhầm Process ID với thông số cần đồng bộ (như Area ID) | Process ID chỉ có ý nghĩa cục bộ trên từng router, không cần khớp giữa các router khác nhau |
| Router không phải DR/BDR bị coi là "lỗi", cần sửa | Nhầm hành vi thiết kế 2-Way với sự cố thực sự | DROTHER giữ 2-Way với nhau là hành vi bình thường trên mạng multi-access, chỉ cần Full với DR/BDR |
| Cost OSPF luôn phản ánh đúng tốc độ thực tế của liên kết | Bỏ qua việc Reference Bandwidth mặc định 100 Mbps đã lỗi thời (mục 6) | Từ Gigabit trở lên, cost mặc định bị bão hòa ở giá trị 1, cần chỉnh `auto-cost reference-bandwidth` nếu muốn phân biệt chính xác |
| DR sẽ tự động đổi khi có router Priority cao hơn tham gia mạng | Bỏ qua cơ chế non-preemptive (mục 5) | DR/BDR chỉ đổi khi router hiện tại down, không tự nhường vai trò dù có ứng viên Priority cao hơn xuất hiện sau |
| Point-to-Point cũng cần bầu DR/BDR giống Ethernet | Áp dụng máy móc cơ chế của Broadcast Multi-Access cho mọi loại mạng | Point-to-Point chỉ có đúng 2 router nên luôn Full-adjacency trực tiếp, không cần DR/BDR (mục 5) |
| `default-information originate` luôn quảng bá default route bất kể router có đường ra thật hay không | Nhầm hành vi mặc định với hành vi khi thêm từ khóa `always` | Mặc định chỉ quảng bá nếu router đã có sẵn default route thật trong bảng; muốn bỏ qua điều kiện này phải thêm `always` (mục 10), và cần cẩn trọng vì rủi ro tương tự case study AS7007 |

## 16. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. OSPF khác Distance Vector ở điểm cốt lõi nào?**
→ OSPF (Link State) xây dựng bản đồ topology đầy đủ (LSDB) giống hệt nhau trên mọi router trong vùng, rồi mỗi router tự chạy thuật toán Dijkstra để tính đường ngắn nhất. Distance Vector chỉ biết hướng đi và khoảng cách từ hàng xóm, không có bản đồ đầy đủ toàn mạng.

**2. Vì sao cần bầu DR/BDR trên mạng multi-access?**
→ Để tránh số lượng adjacency tăng theo bình phương số router (n(n-1)/2) nếu mọi router đều Full-adjacency với nhau. DR/BDR đóng vai trò trung tâm, các router khác chỉ cần Full-adjacency với DR và BDR, giảm tải trao đổi LSA đáng kể.

**3. Trình bày công thức tính Cost của OSPF và vấn đề với Reference Bandwidth mặc định.**
→ Cost = Reference Bandwidth / Bandwidth interface, Reference Bandwidth mặc định 100 Mbps trên Cisco IOS. Vấn đề: từ Gigabit trở lên, cost bị bão hòa ở giá trị 1, không còn phân biệt được các tốc độ liên kết cao hơn — cần chỉnh `auto-cost reference-bandwidth` đồng nhất toàn vùng để khắc phục.

**4. Router ID trùng lặp giữa 2 router OSPF gây ra hậu quả gì?**
→ Các router còn lại nhận LSA từ 2 nguồn khác nhau nhưng router ID giống hệt nhau, khiến LSDB bị ghi đè lẫn nhau liên tục, route trở nên bất ổn định (xuất hiện/biến mất, đổi hướng liên tục) mà không có cảnh báo lỗi rõ ràng.

**5. Vì sao DR/BDR không tự động đổi khi có router Priority cao hơn tham gia mạng sau?**
→ Vì cơ chế bầu chọn DR/BDR của OSPF là non-preemptive — một khi đã bầu xong và ổn định, vai trò DR/BDR chỉ thay đổi khi chính DR/BDR hiện tại gặp sự cố, không bị "giành lại" bởi router mới có Priority cao hơn.

**6. Kể tên ít nhất 4 điều kiện bắt buộc phải khớp để 2 router hình thành OSPF neighbor.**
→ Area ID, Hello/Dead Interval, Router ID (phải khác nhau/duy nhất), Authentication (nếu có bật), và MTU (khuyến nghị khớp, có thể bỏ qua bằng `mtu-ignore`) — thiếu bất kỳ điều kiện bắt buộc nào trong số này, neighbor sẽ không hình thành đúng hoặc không trao đổi được dữ liệu định tuyến.

**7. Vì sao OSPFv3 không dùng lệnh `network` để khai báo interface như OSPFv2?**
→ OSPFv3 kích hoạt trực tiếp trên từng interface bằng lệnh `ipv6 ospf <process-id> area <area>` ngay trong chế độ interface configuration, thay vì dùng wildcard mask để "khớp" interface từ router configuration mode như OSPFv2 — cách này rõ ràng hơn và tránh lỗi tính sai wildcard mask. (Đây là cú pháp legacy; cú pháp khuyến nghị hiện nay là `router ospfv3 <pid>` + `address-family ipv6 unicast` và `ospfv3 <pid> ipv6 area <area>` trên interface, xem lưu ý ở mục 9.)

**8. `default-information originate` và `default-information originate always` khác nhau ở điểm nào?**
→ Không có `always`, router chỉ quảng bá default route vào OSPF nếu chính nó đã có sẵn 1 default route thật trong routing table. Có `always`, router quảng bá default route bất kể có route đó trong bảng hay không — rủi ro là các router khác có thể bị định tuyến nhầm về 1 router thực chất không có đường ra thật.

## 17. Tài liệu tham khảo

- [PingLabz — OSPF Cost: Reference Bandwidth, Manual Overrides, Gotchas](https://www.pinglabz.com/ospf-cost/)
- [PingLabz — OSPF DR and BDR Explained: Designated Router Election Process](https://www.pinglabz.com/ospf-dr-bdr-designated-router/)
- [Cisco — Understand OSPF Neighbor States](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/13685-13.html)
- [Huawei — Case Study: OSPF Routing Loops Caused by Router ID Conflicts](https://support.huawei.com/enterprise/en/doc/EDOC1000177634/e36010e8/case-study-ospf-routing-loops-caused-by-router-id-conflicts)
- [NetworkLessons — OSPF Hello and Dead Interval](https://networklessons.com/ospf/ospf-hello-and-dead-interval)
- [Cisco — IPv6 Routing: OSPFv3 Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-1sg/ip6-route-ospfv3.html)
- [Study CCNA — OSPF Default-Information Originate and the Default Route](https://study-ccna.com/ospf-default-information-originate/)
- [ComputerNetworkingNotes — OSPF Neighborship Building Requirements and States Explained](https://www.computernetworkingnotes.com/ccna-study-guide/ospf-neighborship-building-requirements-and-states-explained.html)

---
**Liên kết**: [[Bai-13_Static-Route|← Bài 13]] · [[Bai-15_EIGRP|Bài 15 - EIGRP →]]
