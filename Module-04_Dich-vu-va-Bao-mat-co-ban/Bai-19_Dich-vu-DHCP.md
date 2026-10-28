---
title: Bài 19 - Dịch vụ DHCP
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — DHCP là dịch vụ nền tảng gần như mọi mạng doanh nghiệp đều dùng, và DHCP Relay là câu hỏi thực chiến rất hay gặp khi có nhiều subnet"
tags: [networking, cisco, dhcp, dora, dhcp-relay, sysadmin]
aliases: [DHCP, DORA, DHCP Relay, DHCP Snooping]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-12_Inter-VLAN-Routing]]"]
last_review: 2026-07-21
---

# Bài 19: Dịch vụ DHCP

> [!important] Tại sao cần file này
> Từ [[Bai-05_Dia-chi-IPv4|Bài 5]] tới nay, mọi ví dụ cấu hình đều giả định IP được **gán tay**. Trong thực tế doanh nghiệp, gán tay IP cho hàng trăm/hàng nghìn thiết bị đầu cuối là không khả thi — **DHCP (Dynamic Host Configuration Protocol)** tự động hóa việc cấp phát IP, giải quyết đúng bài toán này. Đây là 1 trong những dịch vụ mạng cơ bản nhất nhưng lại gắn liền với rất nhiều tình huống thực chiến (từ hết địa chỉ IP, tới tấn công giả mạo DHCP server).

## 1. Mô hình hoạt động và môi trường cần thiết

DHCP hoạt động theo mô hình **Client-Server**: DHCP Client (PC, laptop, điện thoại...) gửi yêu cầu, DHCP Server phản hồi và cấp phát cấu hình IP tự động — không cần người dùng tự nhập IP/Subnet Mask/Gateway/DNS tay. Trên hạ tầng Cisco, DHCP Server có thể chạy trên chính **Router** hoặc **Layer 3 Switch** (đã học SVI ở [[Bai-12_Inter-VLAN-Routing|Bài 12]]), không nhất thiết cần 1 server riêng biệt — dù trong doanh nghiệp lớn, DHCP thường chạy trên server chuyên dụng (Windows Server, Linux) để dễ quản lý tập trung.

**Môi trường cần thiết trước khi triển khai:**

- Tính năng DHCP Server trên Cisco IOS được bật sẵn mặc định (`service dhcp`) — chỉ cần tạo pool là hoạt động ngay, không cần bật thêm; nếu vô tình đã tắt (`no service dhcp`), phải `service dhcp` lại trước khi cấu hình pool.
- Nếu Client và DHCP Server nằm khác subnet (mô hình phổ biến với nhiều VLAN), cần cấu hình định tuyến liên VLAN ([[Bai-12_Inter-VLAN-Routing|Bài 12]]) và DHCP Relay Agent (mục 5) — DHCP không tự vượt qua ranh giới subnet.
- Cần đảm bảo **chỉ có đúng 1 DHCP Server hợp lệ** phục vụ mỗi subnet — sự tồn tại song song của 1 DHCP Server thứ 2 (dù vô tình hay cố ý) gây xung đột cấp phát hoặc rủi ro Rogue DHCP Server (mục 8).

## 2. Phương thức hoạt động — quy trình DORA

```
+--------------+                        +--------------+
| DHCP CLIENT  |                        | DHCP SERVER  |
+--------------+                        +--------------+
        1. DISCOVER (broadcast)   
         -------------------->    

           2. OFFER (unicast)     
         <--------------------    

         3. REQUEST (broadcast)   
         -------------------->    

            4. ACK (unicast)      
         <--------------------    
```

4 bước **DORA**:

1. **Discover**: Client gửi broadcast (chưa có IP nên không thể unicast) để "hỏi" xem có DHCP Server nào trong mạng không.
2. **Offer**: Server phản hồi, "đề nghị" 1 địa chỉ IP cụ thể cùng các thông số đi kèm (Subnet Mask, Gateway, DNS, Lease time).
3. **Request**: Client chính thức "yêu cầu" nhận địa chỉ đã được đề nghị (vẫn gửi broadcast, vì có thể có nhiều DHCP Server cùng phản hồi Offer, Request giúp các server khác biết Client đã chọn server nào).
4. **Acknowledge (ACK)**: Server xác nhận, hoàn tất quá trình cấp phát — Client chính thức được dùng địa chỉ IP đó.

> [!question] Vì sao bước Discover và Request đều dùng broadcast, kể cả khi Client đã biết địa chỉ Server từ bước Offer?
> Vì trong 1 mạng có thể tồn tại **nhiều DHCP Server** cùng lúc (cố ý để dự phòng, hoặc vô tình do rogue DHCP Server — mục 8) — dùng broadcast ở bước Request đảm bảo **mọi** DHCP Server trong mạng đều nhận được thông tin Client đã chọn Offer của server nào, để các server còn lại tự động thu hồi lại IP đã đề nghị (không cấp nhầm IP đó cho Client khác).

Sau khi có IP, Client bước vào trạng thái **Bound** với 2 timer quan trọng: **T1 (mặc định 50% lease time)** — Client unicast trực tiếp về server gốc để gia hạn; **T2 (mặc định 87.5% lease time)** — nếu server gốc không phản hồi, Client broadcast tìm bất kỳ DHCP Server nào khác để gia hạn.

## 3. Các giá trị trong DHCP Pool

| Tham số | Ý nghĩa |
|---|---|
| **Network/Subnet** | Dải mạng mà Pool sẽ cấp phát IP (ví dụ `10.10.10.0/24`) |
| **Default-Router** | Địa chỉ Gateway mà Client sẽ dùng — thường là IP của interface router/SVI trong subnet đó |
| **Default-DNS** | Địa chỉ DNS Server Client sẽ dùng để phân giải tên miền |
| **Lease Time** | Thời gian 1 địa chỉ IP được "cho mượn" trước khi phải gia hạn hoặc trả lại pool |

## 4. Cấu hình DHCP trên Router/Layer 3 Switch

```
Router(config)# ip dhcp excluded-address 10.10.10.1 10.10.10.10
Router(config)# ip dhcp pool KINH-DOANH
Router(dhcp-config)# network 10.10.10.0 255.255.255.0
Router(dhcp-config)# default-router 10.10.10.1
Router(dhcp-config)# dns-server 8.8.8.8 8.8.4.4
Router(dhcp-config)# lease 7
```

> [!question] Vì sao `ip dhcp excluded-address` nên khai báo **trước** khi tạo pool (theo thực hành tốt), và vì sao nó cần thiết?
> Vì mặc định, DHCP Server sẽ cấp phát **toàn bộ** dải IP trong pool, kể cả những IP đã dùng cố định cho thiết bị hạ tầng (như chính địa chỉ Gateway `10.10.10.1`, hoặc IP tĩnh của server nội bộ). Nếu không loại trừ trước, DHCP có thể vô tình cấp phát trùng 1 IP đã đang được dùng cố định cho 1 thiết bị khác, gây xung đột IP (2 thiết bị cùng 1 địa chỉ). Lệnh này khai báo ở chế độ global config vì áp dụng chung cho toàn router, không thuộc riêng 1 pool cụ thể; gõ trước hay sau khi tạo pool đều được (server kiểm tra danh sách loại trừ lúc cấp phát), nhưng nên khai trước để tránh cấp nhầm IP.

## 5. DHCP Relay Agent cho mạng ở xa

Theo mặc định, gói tin DHCP Discover (broadcast) **không đi qua được router** — broadcast bị giới hạn trong 1 broadcast domain (đã học từ [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]). Nếu DHCP Server đặt ở 1 subnet khác với Client (mô hình phổ biến trong doanh nghiệp — 1 DHCP Server tập trung phục vụ nhiều VLAN), cần **DHCP Relay Agent**:

```
Router(config)# interface vlan 20
Router(config-if)# ip helper-address 10.10.10.100
```

Cơ chế hoạt động: router (đóng vai trò Relay Agent) nhận Discover broadcast từ Client, **chuyển đổi thành unicast** gửi thẳng tới DHCP Server đã khai báo, đồng thời gắn thêm trường **giaddr** (Gateway IP Address — chính là IP của interface có `ip helper-address`) vào gói tin. Trường `giaddr` này giúp DHCP Server biết chính xác Client đang ở subnet nào để cấp đúng pool tương ứng, sau đó Server gửi phản hồi ngược lại qua Relay Agent để tới Client.

> [!question] Nếu quên cấu hình `ip helper-address` trên router giữa Client và DHCP Server, hiện tượng gì xảy ra?
> Client sẽ **không bao giờ nhận được IP** — gói Discover broadcast bị chặn ngay tại router (không đi qua được), DHCP Server ở subnet khác không hề biết có Client đang cần cấp IP. Đây là nguyên nhân rất phổ biến của lỗi "PC không tự nhận IP" trong mạng nhiều VLAN có DHCP Server tập trung — cần kiểm tra `ip helper-address` đầu tiên trước khi nghi ngờ chính DHCP Server có vấn đề.

## 6. Quản lý không gian địa chỉ IP

Quy hoạch pool DHCP cần tính toán để tránh cạn kiệt địa chỉ — đặc biệt quan trọng khi lease time ngắn (thiết bị IoT/khách vãng lai xoay vòng nhanh) hoặc subnet nhỏ (VLSM chia quá sát, đã học ở [[Bai-05_Dia-chi-IPv4|Bài 5]]). Thực hành phổ biến: dự trù pool lớn hơn số lượng thiết bị thực tế tối thiểu 20-30% để có khoảng đệm an toàn, đồng thời rà soát định kỳ `show ip dhcp binding` để phát hiện sớm dấu hiệu pool sắp cạn.

**Cấp phát IP cố định cho 1 thiết bị cụ thể qua DHCP (Static Reservation/Binding)** — dùng khi muốn 1 thiết bị (máy in, camera, server) luôn nhận đúng 1 IP qua DHCP thay vì gán tay, nhưng vẫn hưởng các thông số khác (Gateway, DNS) tự động từ pool:

```
Router(config)# ip dhcp pool MAY-IN-P1
Router(dhcp-config)# host 10.10.10.50 255.255.255.0
Router(dhcp-config)# hardware-address 0011.2233.4455
```

`hardware-address` khai báo đúng địa chỉ MAC của thiết bị cần cấp cố định — mỗi lần thiết bị đó xin DHCP, server luôn trả về đúng IP `10.10.10.50` đã khai báo, không đổi qua các lần gia hạn khác nhau. Với thiết bị Windows, dùng `client-identifier` (thêm tiền tố `01` trước MAC) thay vì `hardware-address`, vì Windows DHCP Client gửi định danh khác với chuẩn `hardware-address` mặc định của các hệ điều hành khác.

> [!question] Vì sao nên dùng DHCP Static Reservation thay vì gán IP tĩnh trực tiếp trên chính thiết bị đó?
> Vì Static Reservation vẫn giữ được lợi ích quản lý tập trung của DHCP — mọi thông số (Gateway, DNS, thời gian lease) vẫn do 1 nơi duy nhất (DHCP Server) kiểm soát và có thể thay đổi hàng loạt khi cần, trong khi vẫn đảm bảo đúng thiết bị đó luôn nhận đúng 1 địa chỉ cố định để dễ theo dõi/trỏ tới (ví dụ cấu hình Access Point/Camera theo IP cố định). Gán IP tĩnh trực tiếp trên thiết bị sẽ mất khả năng quản lý tập trung này — muốn đổi DNS cho toàn bộ thiết bị tĩnh phải sửa tay từng cái một.

## 7. Giám sát và xử lý sự cố

```
Router# show ip dhcp binding
Router# show ip dhcp pool
Router# show ip dhcp conflict
Router# show ip dhcp server statistics
```

`show ip dhcp binding` liệt kê toàn bộ IP đã cấp phát cùng MAC tương ứng và thời gian lease còn lại — lệnh đầu tiên cần chạy khi nghi ngờ vấn đề cấp phát IP. `show ip dhcp conflict` hiển thị các địa chỉ IP bị phát hiện xung đột (server tự kiểm tra bằng ping/ARP trước khi cấp phát).

## 8. Rủi ro bảo mật liên quan DHCP — DHCP Starvation và Rogue DHCP Server

Hai kỹ thuật tấn công phổ biến liên quan tới DHCP (phòng chống chi tiết bằng DHCP Snooping sẽ học ở Bài 27):

- **DHCP Starvation**: kẻ tấn công gửi hàng loạt Discover với địa chỉ MAC giả mạo khác nhau, cố tình "vét cạn" toàn bộ pool DHCP hợp lệ trước khi thiết bị thật kịp xin IP.
- **Rogue DHCP Server**: sau khi pool hợp lệ cạn kiệt (hoặc đơn giản là chạy song song), kẻ tấn công dựng 1 DHCP Server giả, cấp phát Gateway/DNS trỏ về thiết bị của kẻ tấn công — biến đây thành 1 dạng tấn công Man-in-the-Middle (toàn bộ traffic Client đi qua "gateway giả" trước khi ra ngoài).

> [!question] Vì sao Rogue DHCP Server nguy hiểm hơn nhiều so với việc chỉ đơn thuần "gây rối" cấp sai IP?
> Vì Rogue DHCP Server không chỉ cấp sai IP — nó có thể cấp **đúng thông số Gateway và DNS trỏ về thiết bị của kẻ tấn công**. Một khi Client dùng cấu hình đó, mọi traffic của Client (bao gồm cả traffic tưởng chừng đã mã hóa ở tầng ứng dụng) đều đi qua thiết bị kẻ tấn công trước khi ra ngoài — mở đường cho nghe lén, chỉnh sửa traffic, hoặc chuyển hướng DNS tới các trang giả mạo mà người dùng khó nhận ra vì địa chỉ IP/domain hiển thị vẫn có vẻ hợp lệ.

## 9. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 1 Router/L3 Switch, 2 VLAN, 1 PC ở mỗi VLAN.

1. Cấu hình DHCP Pool cho VLAN 10 theo mục 4, xác nhận PC ở VLAN 10 tự nhận đúng IP/Gateway/DNS.
2. Đặt DHCP Server (pool) chỉ phục vụ VLAN 10, nhưng thử để PC ở VLAN 20 xin DHCP — xác nhận PC ở VLAN 20 **không** nhận được IP do broadcast bị chặn tại router.
3. Thêm `ip helper-address` trên SVI VLAN 20 trỏ về DHCP Server theo mục 5, thử lại — xác nhận PC VLAN 20 giờ đã nhận IP đúng pool của VLAN 20 (cần tạo thêm pool riêng cho VLAN 20 trước).
4. Chạy `show ip dhcp binding`, đối chiếu danh sách IP đã cấp khớp với các PC đã test.

**Kết quả kỳ vọng**: hiểu rõ vì sao DHCP Relay cần thiết trong mạng nhiều VLAN, và thành thạo cấu hình + giám sát DHCP cơ bản.

## 10. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| DHCP Server tự động biết không cấp phát các IP đã dùng cố định | Bỏ qua việc phải khai báo tường minh | Cần `ip dhcp excluded-address` để loại trừ dải IP đã dùng cố định (mục 4) |
| Broadcast Discover luôn tới được DHCP Server dù ở subnet khác | Bỏ qua giới hạn broadcast domain của router | Cần DHCP Relay Agent (`ip helper-address`) nếu Server và Client khác subnet (mục 5) |
| T1 và T2 là 2 cơ chế độc lập, không liên quan nhau | Nhầm lẫn về trình tự gia hạn lease | T1 (50%) unicast về server gốc trước, chỉ khi không thành công mới tới T2 (87.5%) broadcast tìm server khác (mục 2) |
| Chỉ cần quan tâm đủ IP trong pool, không cần lo về bảo mật DHCP | Bỏ qua rủi ro Rogue DHCP Server (mục 8) | Rogue DHCP có thể dẫn tới Man-in-the-Middle nghiêm trọng hơn nhiều so với việc thiếu IP, cần biện pháp như DHCP Snooping (Bài 27) |

## 11. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Trình bày 4 bước của quy trình DORA.**
→ Discover (Client broadcast tìm DHCP Server) → Offer (Server đề nghị 1 IP cụ thể) → Request (Client broadcast xác nhận chọn Offer đó, để các server khác biết mà thu hồi Offer của mình) → Acknowledge (Server xác nhận hoàn tất cấp phát).

**2. Vì sao Request ở bước 3 vẫn dùng broadcast dù Client đã biết địa chỉ Server cụ thể từ Offer?**
→ Vì có thể có nhiều DHCP Server cùng phản hồi Offer; broadcast ở bước Request đảm bảo mọi Server trong mạng đều biết Client đã chọn Offer của server nào, để các Server còn lại thu hồi Offer chưa được chọn, tránh cấp trùng IP.

**3. DHCP Relay Agent giải quyết vấn đề gì, và cấu hình bằng lệnh nào?**
→ Giải quyết việc gói Discover broadcast không đi qua được router khi DHCP Server đặt ở subnet khác Client. Cấu hình bằng `ip helper-address <IP DHCP Server>` trên interface/SVI của subnet chứa Client.

**4. `ip dhcp excluded-address` dùng để làm gì, và vì sao nên khai báo trước khi tạo pool?**
→ Loại trừ 1 dải IP khỏi việc cấp phát tự động, thường dùng cho các địa chỉ đã gán cố định (Gateway, server tĩnh). Khai báo ở chế độ global vì lệnh này áp dụng chung toàn router, không thuộc về 1 pool cụ thể nào; gõ trước hay sau pool đều được (server kiểm tra lúc cấp phát), nhưng nên khai trước theo thực hành tốt.

**5. Rogue DHCP Server nguy hiểm như thế nào so với DHCP Starvation đơn thuần?**
→ DHCP Starvation chỉ làm cạn pool hợp lệ, gây gián đoạn dịch vụ. Rogue DHCP Server nguy hiểm hơn vì có thể cấp Gateway/DNS trỏ về thiết bị kẻ tấn công, biến thành tấn công Man-in-the-Middle khiến toàn bộ traffic Client đi qua kẻ tấn công trước khi ra ngoài.

## 12. Tài liệu tham khảo

- [NetsTuts — DHCP – Dynamic Host Configuration Protocol Overview: DORA Process, Scope, Relay](https://www.netstuts.com/dhcp)
- [Cisco — Configuring the Cisco IOS DHCP Relay Agent](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/15-sy/dhcp-15-sy-book/dhcp-relay-agent.html)
- [Connected — How DHCP Lease Works (T1/T2 timers)](https://www.connected.app/library/how-dhcp-lease-works-6gw35ms)
- [DeepStrike — What Is DHCP Starvation? How Attackers Exhaust IP Pools](https://deepstrike.io/blog/what-is-dhcp-starvation)

---
**Liên kết**: [[Bai-18_NTP-va-Syslog|← Bài 18]] · [[Bai-20_NAT|Bài 20 - NAT →]]
