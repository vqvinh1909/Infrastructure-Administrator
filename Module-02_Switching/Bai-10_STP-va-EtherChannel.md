---
title: Bài 10 - STP và EtherChannel
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — STP là kiến thức nền bắt buộc để thiết kế mạng Layer 2 có dự phòng an toàn"
tags: [networking, cisco, stp, rstp, etherchannel, lacp, pagp, switching, sysadmin]
aliases: [Spanning Tree Protocol, EtherChannel, LACP, PAgP, RSTP]
related: ["[[Bai-09_Trunking-DTP-VTP]]", "[[Bai-02_Mo-hinh-OSI-TCPIP]]"]
last_review: 2026-07-21
---

# Bài 10: STP và EtherChannel

> [!important] Tại sao cần file này
> [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]] (mục 5.3) đã nói: lớp Core luôn cần **2 switch dự phòng nhau**. Nhưng nối dư thừa đường truyền ở Lớp 2 lại tạo ra một vấn đề nghiêm trọng khác — **vòng lặp (loop)**. Case study broadcast storm tại NERC đã học ở [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]] chính là hậu quả thật của một switching loop không được kiểm soát. STP là giao thức giải quyết đúng mâu thuẫn này: vừa giữ được đường dự phòng vật lý, vừa đảm bảo logic mạng luôn là hình cây (tree) không vòng lặp.

## 1. Vì sao xuất hiện vòng lặp khi có đường dự phòng

```
        SW1
       /    \
      /      \
   SW2 ------ SW3
```

Khi 3 switch nối thành hình tam giác để dự phòng cho nhau, một frame broadcast gửi từ SW2 sẽ được SW1 và SW3 cùng nhận, rồi mỗi switch lại tiếp tục gửi frame đó sang switch còn lại — frame **nhân bản vô hạn** và chạy vòng quanh 3 switch mãi mãi, không có TTL như ở Lớp 3 để tự triệt tiêu (nhắc lại: Ethernet Lớp 2 không có khái niệm TTL). Đây chính là cơ chế kỹ thuật đứng sau case study broadcast storm ở Bài 2 — CPU switch bị quá tải xử lý cùng 1 frame lặp lại liên tục, bảng MAC "loạn" vì cùng 1 MAC nguồn xuất hiện ở nhiều cổng khác nhau trong tích tắc (MAC flapping).

## 2. Cơ chế hoạt động của STP (802.1D)

STP giải quyết vấn đề trên bằng cách **tính toán và chủ động khóa (block)** một số cổng dư thừa, biến topology vật lý có vòng lặp thành topology logic hình cây (tree) không vòng lặp — nhưng vẫn giữ nguyên đường cáp vật lý để tự động kích hoạt lại nếu đường chính hỏng.

**Bước 1 — Bầu chọn Root Bridge:** mọi switch trao đổi BPDU (Bridge Protocol Data Unit) chứa Bridge ID (Priority + MAC address). Switch có **Bridge ID nhỏ nhất** trở thành Root Bridge — điểm tham chiếu trung tâm để tính toán toàn bộ cây.

**Bước 2 — Xác định vai trò từng cổng trên các switch còn lại:**

| Vai trò cổng | Ý nghĩa |
|---|---|
| **Root Port** | Cổng có đường đi (cost) rẻ nhất về Root Bridge — mỗi switch (trừ Root Bridge) có đúng 1 Root Port |
| **Designated Port** | Cổng "đại diện" tốt nhất cho 1 đoạn mạng (segment) — chuyển tiếp traffic bình thường |
| **Blocked / Non-Designated Port** | Cổng dư thừa, bị khóa để phá vòng lặp — vẫn nhận BPDU để giám sát nhưng không chuyển tiếp traffic dữ liệu |

**Chi phí đường đi (Port Cost)** — càng tốc độ cao, cost càng thấp (đường "rẻ" hơn được ưu tiên):

| Tốc độ | Cost thang ngắn (short, 802.1D-1998 — IOS cũ, Packet Tracer, lab) |
|---|---|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

> **Lưu ý cập nhật:** bảng trên là thang đo ngắn (short). Catalyst 9300 chạy IOS-XE mặc định dùng thang dài (long path-cost): 10 Mbps = 2.000.000, 100 Mbps = 200.000, 1 Gbps = 20.000, 10 Gbps = 2.000. Khi so sánh cost giữa các switch, phải bảo đảm cùng một thang đo.

**4 trạng thái cổng hoạt động của STP cổ điển (802.1D)** (chuẩn còn định nghĩa thêm trạng thái thứ 5 là Disabled cho cổng bị tắt) và thời gian chuyển đổi mặc định:

```
Blocking (20s Max Age) -> Listening (15s Forward Delay) -> Learning (15s Forward Delay) -> Forwarding
```

Tổng thời gian hội tụ STP cổ điển có thể lên đến **~50 giây** — đây chính là nhược điểm lớn nhất khiến RSTP ra đời (mục 3).

> **Lưu ý cập nhật:** 802.1w đã được gộp vào 802.1D-2004, và từ 802.1Q-2014 các giao thức STP/RSTP/MSTP đều nằm trong 802.1Q. Ngoài ra còn có MSTP (802.1s) cho phép ánh xạ nhiều VLAN vào một số instance STP.

## 3. RSTP — Rapid Spanning Tree Protocol (802.1w)

RSTP cải tiến STP cổ điển với thời gian hội tụ nhanh hơn nhiều (thường dưới vài giây thay vì ~50 giây) nhờ bổ sung 2 vai trò cổng mới và rút gọn còn 3 trạng thái:

| | STP (802.1D) | RSTP (802.1w) |
|---|---|---|
| Số trạng thái cổng | 4 (Blocking, Listening, Learning, Forwarding) | 3 (Discarding, Learning, Forwarding) |
| Vai trò cổng bổ sung | Không có | **Alternate Port** (dự phòng cho Root Port), **Backup Port** (dự phòng cho Designated Port) |
| Thời gian hội tụ | ~50 giây | Thường dưới 6 giây, có thể gần như tức thời trong nhiều trường hợp |
| Cơ chế | Chờ hết timer mới chuyển trạng thái | Trao đổi tường minh (handshake) giữa các switch để chuyển nhanh sang Forwarding |

> [!question] Alternate Port khác Blocked Port của STP cổ điển ở điểm nào?
> Về hành vi chuyển tiếp traffic thì giống nhau (đều không chuyển tiếp dữ liệu). Khác biệt là RSTP **đã tính toán sẵn** Alternate Port là "phương án dự phòng ngay lập tức" cho Root Port — khi Root Port hỏng, Alternate Port chuyển sang Forwarding gần như tức thì mà không cần tính toán lại từ đầu như STP cổ điển phải trải qua Listening → Learning.

## 4. Per-VLAN STP (PVST+/Rapid PVST+)

Chuẩn STP/RSTP gốc (802.1D/802.1w) chỉ chạy **1 instance duy nhất cho toàn bộ mạng**, dù có bao nhiêu VLAN. Cisco mở rộng thành **PVST+ (STP)** và **Rapid PVST+ (RSTP)** — chạy **1 instance STP riêng cho mỗi VLAN**, cho phép:

- Mỗi VLAN có Root Bridge khác nhau, tận dụng được cân bằng tải (VLAN 10 dùng SW1 làm Root, VLAN 20 dùng SW2 làm Root) thay vì tất cả VLAN cùng dồn traffic qua 1 Root Bridge duy nhất.
- Đây là chế độ **mặc định** trên switch Cisco hiện đại (`rapid-pvst`).

## 5. Cấu hình STP

```
Switch(config)# spanning-tree mode rapid-pvst
Switch(config)# spanning-tree vlan 10 root primary
Switch(config)# spanning-tree vlan 20 root secondary
```

`root primary` tự động đặt priority = 24576 (hoặc thấp hơn giá trị Root hiện tại nếu cần, theo bội số 4096 gần nhất). `root secondary` đặt priority = 28672 — trở thành Root Bridge dự phòng nếu Root chính hỏng. Có thể đặt tay chính xác bằng:

```
Switch(config)# spanning-tree vlan 10 priority 4096
```

## 6. Verify STP

```
Switch# show spanning-tree vlan 10
Switch# show spanning-tree summary
```

`show spanning-tree vlan 10` hiển thị: Root Bridge hiện tại là switch nào, vai trò từng cổng trên chính switch đang xem (Root/Designated/Blocked), và cost tương ứng — đây là lệnh đầu tiên cần chạy khi nghi ngờ có vấn đề về loop hoặc thiết kế dự phòng không như mong đợi.

## 7. EtherChannel — gộp nhiều liên kết vật lý thành 1 liên kết logic

EtherChannel gộp **2 đến 8 cổng vật lý** (tối đa 8 cổng active; riêng LACP cho cấu hình tới 16 cổng gồm 8 active + 8 hot-standby) giữa 2 switch thành **1 liên kết logic duy nhất (Port-Channel)** — mang lại 2 lợi ích cùng lúc:

- **Tăng băng thông**: nhiều cổng Gigabit gộp lại cho tổng băng thông cao hơn (dù không cộng tuyến tính hoàn toàn do cơ chế phân bổ traffic theo hash).
- **Dự phòng KHÔNG bị STP khóa bớt**: đây là điểm khác biệt cốt lõi so với việc chỉ nối nhiều cáp vật lý riêng lẻ — STP nhìn Port-Channel như **1 cổng logic duy nhất**, nên không coi các cổng thành viên là đường dư thừa cần khóa. Nếu không dùng EtherChannel mà chỉ nối 2-3 cáp riêng giữa cùng 2 switch, STP sẽ khóa bớt gần hết, chỉ giữ lại 1 cáp hoạt động — lãng phí băng thông của các cáp còn lại.

## 8. Các mode cấu hình EtherChannel

| Giao thức | Mode | Hành vi |
|---|---|---|
| **Static (không đàm phán)** | `on` | Ép cứng thành EtherChannel ngay lập tức, không kiểm tra tương thích — rủi ro nếu 2 đầu cấu hình không khớp |
| **PAgP** (độc quyền Cisco) | `desirable` | Chủ động đề nghị hình thành EtherChannel |
| | `auto` | Bị động, chỉ phản hồi nếu đầu kia chủ động đề nghị |
| **LACP** (chuẩn mở IEEE 802.1AX, trước đây là 802.3ad) | `active` | Chủ động đề nghị hình thành EtherChannel |
| | `passive` | Bị động, chỉ phản hồi nếu đầu kia chủ động đề nghị |

**Nguyên tắc tương thích**: PAgP và LACP **không tương thích với nhau** — 2 đầu phải cùng dùng 1 giao thức. Khuyến nghị thực tế hiện nay: luôn dùng **LACP active-active** cho triển khai mới vì là chuẩn mở (không phụ thuộc hãng) và có đàm phán tường minh giúp phát hiện lỗi cấu hình sai trước khi hình thành kênh — khác với `on` (static) hoàn toàn không kiểm tra gì.

## 9. Chế độ hoạt động của Port-Channel

Sau khi hình thành, Port-Channel hoạt động như 1 interface logic bình thường và có thể cấu hình như bất kỳ cổng vật lý nào đã học:

- **Access**: gán vào 1 VLAN cụ thể (giống mục 4, [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8]]).
- **Trunk**: mang nhiều VLAN (giống mục 5, [[Bai-09_Trunking-DTP-VTP|Bài 9]]).
- **Layer 3**: nếu switch hỗ trợ Layer 3, Port-Channel có thể gán thẳng địa chỉ IP để định tuyến (liên quan Bài 12 — Inter-VLAN Routing).

## 10. Cấu hình EtherChannel

```
Switch(config)# interface range gigabitEthernet 0/23 - 24
Switch(config-if-range)# channel-group 1 mode active
Switch(config-if-range)# exit
Switch(config)# interface port-channel 1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
```

Lưu ý quan trọng: **mọi cổng thành viên phải có cấu hình giống hệt nhau** trước khi gộp (cùng speed, cùng duplex, cùng mode access/trunk, cùng VLAN allowed nếu là trunk) — nếu không, EtherChannel sẽ không hình thành hoặc hình thành không ổn định.

## 11. Verify EtherChannel

```
Switch# show etherchannel summary
Switch# show interfaces port-channel 1
```

`show etherchannel summary` hiển thị trạng thái từng cổng thành viên — ký hiệu `P` (bundled, hoạt động bình thường trong channel) là trạng thái mong muốn; các ký hiệu khác (`I`, `D`, `H`...) đều thể hiện cổng đó **không** tham gia được vào channel, thường do cấu hình không khớp giữa các cổng thành viên (mục 10).

## 12. Các lỗi thường gặp liên quan đến STP và EtherChannel và cách xử lý

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Một cổng bất ngờ chuyển sang Blocking dù không rõ có vòng lặp nào | Thay đổi topology hoặc priority khiến STP tính lại vai trò cổng, cổng đó trở thành Non-Designated (mục 2) | `show spanning-tree vlan X` xem Root Bridge hiện tại và vai trò/cost từng cổng để xác định vì sao cổng đó thua trong tính toán |
| Mạng gián đoạn ngắn (vài chục giây) mỗi khi thêm/rút 1 switch hoặc 1 cáp | STP đang hội tụ lại (bình thường nếu là STP cổ điển, ~50s theo mục 2); nếu dùng RSTP mà vẫn chậm như vậy, khả năng có switch cũ chỉ hỗ trợ 802.1D buộc cả đoạn mạng "rớt" về chế độ tương thích STP cổ điển | `show spanning-tree summary` kiểm tra mode đang chạy thực tế (PVST/Rapid-PVST) trên toàn bộ switch liên quan, xác định switch nào chưa hỗ trợ RSTP |
| `show etherchannel summary` không báo trạng thái `P` cho các cổng thành viên | Cấu hình các cổng thành viên không khớp nhau (khác speed/duplex/mode access-trunk/VLAN allowed), hoặc 1 đầu dùng PAgP còn đầu kia dùng LACP (mục 8) | Đối chiếu `show running-config interface` từng cổng thành viên ở cả 2 đầu, đảm bảo giống hệt nhau và cùng 1 giao thức đàm phán |
| EtherChannel đã lên nhưng traffic dồn gần hết vào 1 cổng thành viên | Thuật toán load-balancing hash (mặc định thường theo địa chỉ nguồn/đích) cho ra cùng 1 kết quả với phần lớn traffic thực tế trong mạng | Kiểm tra/đổi tiêu chí bằng `port-channel load-balance`, chọn tiêu chí phù hợp hơn với đặc điểm traffic thực tế (ví dụ src-dst-ip thay vì src-mac) |
| Sau khi gộp EtherChannel, STP vẫn khóa một vài cổng trong nhóm | Một số cổng chưa thực sự tham gia channel thành công (không ở trạng thái `P`), STP vẫn coi các cổng đó là cổng vật lý riêng lẻ, độc lập với Port-Channel | `show etherchannel summary` xác nhận toàn bộ cổng thành viên dự kiến đều ở trạng thái `P`, xử lý cổng lỗi theo hàng phía trên |
| Root Bridge thực tế không đúng switch đã đặt `root primary` | Có switch khác trong mạng đang giữ priority thấp hơn giá trị vừa đặt (thường do ai đó từng đặt priority tay từ trước), hoặc lệnh áp nhầm VLAN | `show spanning-tree vlan X` xem Bridge ID của Root hiện tại, so sánh priority các switch liên quan bằng `show spanning-tree summary` trên từng switch |

## 13. Case study — nhìn lại broadcast storm ở Bài 2 dưới góc nhìn STP

> [!example] STP chính là biện pháp phòng ngừa cho case study đã học ở Bài 2
> Sự cố broadcast storm trong báo cáo NERC ([[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]], mục 6) xảy ra vì một switching loop không được kiểm soát khiến CPU switch quá tải gần 100%, mất khả năng quản trị, gói tin mất gần như hoàn toàn — đội vận hành phải cô lập thủ công từng đoạn mạng để tìm ra cổng gây loop. Với STP (hoặc RSTP) được bật đúng cách trên toàn bộ switch liên quan, rủi ro này được **giảm đáng kể**: ngay khi vòng lặp vật lý hình thành, STP sẽ tự động phát hiện và khóa cổng dư thừa trước khi broadcast storm kịp bùng phát. Đây là lý do STP gần như luôn được bật mặc định trên switch Cisco (không giống VTP ở Bài 9, thứ mà nhiều tổ chức chủ động tắt) — rủi ro của loop không kiểm soát được đánh giá là nghiêm trọng hơn nhiều so với bất kỳ chi phí vận hành nào của STP.

> **Lưu ý cập nhật:** STP vẫn có thể thất bại và gây storm khi mất BPDU, liên kết một chiều, cổng PortFast bị cắm vòng hoặc cấu hình `bpdufilter` sai. Vì vậy cần các cơ chế bảo vệ: PortFast (cho cổng access), BPDU Guard (tắt cổng PortFast khi nhận BPDU), Root Guard (chặn switch lạ giành vai trò Root), Loop Guard (chống cổng chuyển Forwarding nhầm khi mất BPDU) và UDLD (phát hiện liên kết một chiều).

## 14. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 3 switch nối hình tam giác như mục 1.

1. Nối 3 switch thành hình tam giác, bật `spanning-tree mode rapid-pvst` (thường đã mặc định), quan sát bằng `show spanning-tree vlan 1` để xác định switch nào là Root Bridge và cổng nào bị Blocking trên 2 switch còn lại.
2. Đổi priority của 1 switch bất kỳ thành giá trị thấp nhất, xác nhận switch đó trở thành Root Bridge mới và vai trò các cổng blocking thay đổi tương ứng.
3. Giữa 2 switch bất kỳ, thay vì nối 1 cáp, nối 2 cáp riêng biệt — quan sát STP tự động khóa bớt 1 trong 2 cáp đó (`show spanning-tree` sẽ chỉ ra cổng bị Blocking).
4. Gỡ bỏ 2 cáp riêng ở bước 3, thay bằng cấu hình EtherChannel LACP active-active theo mục 10 cho đúng 2 cổng đó — xác nhận `show etherchannel summary` báo cả 2 cổng đều ở trạng thái `P` (bundled), và STP giờ đây coi đây là 1 cổng logic duy nhất, không khóa cổng nào.

**Kết quả kỳ vọng**: chứng minh được bằng thực hành sự khác biệt giữa "2 cáp riêng bị STP khóa bớt 1" và "2 cáp gộp EtherChannel được dùng đồng thời cả 2" — đây là điểm hay bị nhầm lẫn nhất giữa STP và EtherChannel.

## 15. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Nối nhiều cáp dự phòng giữa 2 switch tự động tăng băng thông | Bỏ qua việc STP sẽ khóa bớt cáp dư thừa nếu không gộp EtherChannel | Nhiều cáp riêng lẻ (không EtherChannel) chỉ có 1 cáp thực sự truyền dữ liệu, các cáp còn lại bị Blocking |
| RSTP là một giao thức hoàn toàn khác, không liên quan STP | Nhầm lẫn vì tên gọi và cơ chế cải tiến khác nhiều | RSTP (802.1w) là bản nâng cấp trực tiếp của STP (802.1D), cùng mục tiêu chống loop nhưng hội tụ nhanh hơn nhiều |
| PAgP và LACP có thể dùng lẫn 2 đầu miễn là cả 2 đều "EtherChannel" | Nhầm lẫn khái niệm chung (EtherChannel) với giao thức đàm phán cụ thể | 2 đầu bắt buộc cùng 1 giao thức (cùng PAgP hoặc cùng LACP), không trộn lẫn được |
| STP luôn nên tắt để tăng hiệu năng mạng vì "gây độ trễ" | Đánh giá thấp rủi ro loop/broadcast storm khi tắt STP | Độ trễ hội tụ của STP/RSTP là chi phí rất nhỏ so với rủi ro sập toàn mạng nếu xảy ra loop không kiểm soát (case study mục 13) |

## 16. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao mạng có đường dự phòng vật lý (redundant link) lại cần STP?**
→ Vì đường dự phòng ở Lớp 2 tạo ra vòng lặp vật lý; Ethernet không có TTL để tự triệt tiêu frame như Lớp 3, nên broadcast/traffic không rõ đích sẽ nhân bản vô hạn (broadcast storm). STP tính toán và khóa cổng dư thừa để biến topology vật lý có loop thành cây logic không loop, vẫn giữ đường dự phòng để kích hoạt lại khi cần.

**2. Root Bridge được bầu chọn dựa trên tiêu chí gì?**
→ Bridge ID nhỏ nhất (kết hợp Priority và địa chỉ MAC) — switch có Bridge ID nhỏ nhất trở thành Root Bridge, là điểm tham chiếu để toàn mạng tính toán vai trò các cổng còn lại.

**3. RSTP cải tiến gì so với STP cổ điển?**
→ Rút gọn từ 4 trạng thái cổng xuống 3, bổ sung vai trò Alternate Port và Backup Port giúp có sẵn phương án dự phòng, chuyển sang Forwarding gần như tức thì thay vì phải chờ qua Listening/Learning như STP cổ điển — giảm thời gian hội tụ từ ~50 giây xuống dưới vài giây.

**4. EtherChannel giải quyết vấn đề gì mà STP đơn thuần không giải quyết được khi có nhiều cáp dự phòng?**
→ Nếu chỉ nối nhiều cáp vật lý riêng lẻ, STP sẽ khóa bớt để tránh loop, chỉ 1 cáp thực sự hoạt động. EtherChannel gộp các cáp đó thành 1 cổng logic duy nhất — STP không còn coi chúng là đường dư thừa cần khóa, cho phép dùng đồng thời toàn bộ băng thông của tất cả cáp thành viên.

**5. Vì sao không thể trộn 1 đầu PAgP với 1 đầu LACP trên cùng EtherChannel?**
→ Vì đây là 2 giao thức đàm phán độc lập, không tương thích cú pháp bản tin với nhau — cả 2 đầu channel bắt buộc dùng chung 1 giao thức đàm phán (cùng PAgP hoặc cùng LACP), khác giao thức sẽ không hình thành được channel.

## 17. Tài liệu tham khảo

- [ComputingForGeeks — Configure Spanning Tree (Rapid PVST+) on Cisco](https://computingforgeeks.com/cisco-spanning-tree-protocol-configuration/)
- [Firewall.cx — Spanning Tree Protocol: Rapid STP Port Costs - Port States](https://www.firewall.cx/networking/network-protocols/spanning-tree-protocol/spanning-tree-protocol-port-costs-states.html)
- [NetworkDirection — EtherChannel Configuration: LACP, PAgP, and Static on Cisco IOS](https://networkdirection.net/articles/routingandswitching/etherchannelsandlags/etherchannelconfiguration/)

---
**Liên kết**: [[Bai-09_Trunking-DTP-VTP|← Bài 9]] · Bài 11 - Giới thiệu về định tuyến → (đang soạn, Module 3)
