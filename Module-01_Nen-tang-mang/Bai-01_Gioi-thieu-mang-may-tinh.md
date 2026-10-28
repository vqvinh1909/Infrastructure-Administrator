---
title: Bài 1 - Giới thiệu về Mạng máy tính
type: Foundation
difficulty: Beginner
lab_required: true
exam: "Sysadmin Junior — kiến thức nền bắt buộc trước khi cấu hình thiết bị Cisco"
tags: [networking, fundamentals, topology, lan-man-wan, cloud, sysadmin]
aliases: [Gioi thieu mang may tinh, Network Fundamentals]
related: ["[[Bai-02_Mo-hinh-OSI-TCPIP]]"]
last_review: 2026-07-21
---

# Bài 1: Giới thiệu về Mạng máy tính

> [!important] Tại sao cần file này
> Trước khi chạm vào một switch/router thật, sysadmin phải trả lời được: hệ thống mạng gồm những gì, được tổ chức theo mô hình nào, và mô hình đó ảnh hưởng thế nào đến chi phí, độ tin cậy, khả năng mở rộng. Bỏ qua nền tảng này để nhảy thẳng vào cấu hình thiết bị là lý do phổ biến nhất khiến người mới thiết kế sai ngay từ đầu.

## 1. Mạng máy tính là gì và tại sao tồn tại

Mạng máy tính (Computer Network) là tập hợp các thiết bị được kết nối với nhau qua phương tiện truyền dẫn nhằm **chia sẻ tài nguyên** (dữ liệu, máy in, kết nối Internet) và **giao tiếp**. Bản chất công việc của một hệ thống mạng chỉ có một: đưa dữ liệu từ điểm A đến điểm B một cách tin cậy. Mọi khái niệm khác (VLAN, routing, ACL...) ở các bài sau đều xoay quanh việc làm cho quá trình đó nhanh hơn, an toàn hơn, hoặc có tổ chức hơn.

Một hệ thống mạng gồm 3 nhóm thành phần:

- **End device (thiết bị đầu cuối)**: nơi dữ liệu được tạo ra hoặc tiêu thụ — PC, server, máy in, điện thoại IP, camera IP.
- **Intermediary device (thiết bị trung gian)**: chuyển tiếp dữ liệu giữa các end device — switch, router, access point, firewall. Đây là nhóm thiết bị mà network/system admin trực tiếp cấu hình hàng ngày.
- **Network media (phương tiện truyền dẫn)**: đường truyền vật lý hoặc không dây — cáp UTP, cáp quang, sóng Wi-Fi (chi tiết ở Bài 3).

## 2. Topology — Bus, Ring, Star

Topology mô tả cách các thiết bị được kết nối vật lý với nhau. Đây là quyết định thiết kế đầu tiên ảnh hưởng trực tiếp đến khả năng chịu lỗi (fault tolerance) của toàn mạng.

```
BUS                          RING                          STAR
PC1  PC2  PC3  PC4            PC1 ---- PC2                        +--------+
 |    |    |    |               |        |                        | Switch |
 +----+----+----+              PC4 ---- PC3                       +--------+
  (cáp trục dùng chung)     (vòng khép kín, 1 hoặc 2 chiều)       /   |   \
                                                                PC1  PC2  PC3
```

| Topology | Cơ chế | Điểm chịu lỗi | Hiện trạng dùng thực tế |
|---|---|---|---|
| **Bus** | Mọi thiết bị dùng chung một cáp trục | 1 điểm đứt cáp → sập cả đoạn mạng | Gần như không còn dùng trong LAN doanh nghiệp |
| **Ring** | Mỗi thiết bị nối đúng 2 láng giềng, dữ liệu đi vòng | 1 thiết bị/đoạn cáp hỏng có thể ngắt cả vòng (trừ ring kép) | Hiếm ở LAN, còn ở backbone WAN một số nhà mạng (SONET/SDH) |
| **Star** | Mọi thiết bị nối về 1 switch trung tâm | Chỉ thiết bị đó bị ảnh hưởng, switch trung tâm là SPOF | Chuẩn phổ biến nhất của LAN Ethernet hiện nay |

> [!question] Vì sao Star trở thành chuẩn thay vì Bus?
> Vì khả năng cô lập sự cố (fault isolation): khi một cáp hoặc một PC hỏng, switch vẫn cô lập vấn đề ở đúng cổng đó, không kéo sập các thiết bị còn lại. Đây cũng là lý do khi troubleshoot LAN, việc đầu tiên luôn là kiểm tra đúng cổng switch của thiết bị gặp sự cố, không phải nghi ngờ toàn mạng. Đánh đổi: switch trung tâm trở thành single point of failure — lý do các bài sau (VLAN, EtherChannel, STP) đều xoay quanh việc thêm redundancy cho lớp switch.

> [!example] Case study — Facebook outage 04/10/2021 (~7 giờ)
> Từ 15:40 đến 22:45 UTC ngày 04/10/2021, Facebook/Instagram/WhatsApp mất kết nối toàn cầu hơn 7 tiếng. Nguyên nhân: một lệnh bảo trì chạy để đánh giá dung lượng backbone đã vô tình rút toàn bộ route BGP quảng bá các data center của Facebook ra khỏi Internet, kéo theo DNS server của Facebook (vẫn hoạt động bình thường bên trong) trở nên không thể truy cập được từ bên ngoài. Bài học ở mức nền tảng: dù thiết kế đã có dự phòng ở quy mô backbone khổng lồ, **một điểm lỗi cấu hình duy nhất vẫn có thể vô hiệu hóa toàn bộ hệ thống** nếu không có cơ chế an toàn (safeguard) chặn thao tác nguy hiểm — cùng nguyên lý với việc một switch trung tâm không dự phòng có thể sập cả mạng Star, chỉ khác ở quy mô. Nguồn: [Engineering at Meta](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/), [Wikipedia](https://en.wikipedia.org/wiki/2021_Facebook_outage).

## 3. LAN, MAN, WAN — ranh giới trách nhiệm, không chỉ là phạm vi

| Loại | Phạm vi | Ví dụ | Ai sở hữu hạ tầng |
|---|---|---|---|
| **LAN** | Một tòa nhà, một văn phòng | Mạng nội bộ công ty | Doanh nghiệp tự đầu tư, toàn quyền cấu hình |
| **MAN** | Một thành phố/khu vực | Kết nối các chi nhánh ngân hàng cùng thành phố | Doanh nghiệp lớn hoặc thuê hạ tầng ISP |
| **WAN** | Liên tỉnh, liên quốc gia | Internet, kết nối trụ sở chính ↔ chi nhánh nước ngoài | Nhà cung cấp dịch vụ viễn thông (ISP) |

Ranh giới quan trọng nhất về mặt vận hành: **LAN do đội IT toàn quyền kiểm soát**, còn **MAN/WAN thường đi thuê từ ISP**. Hệ quả thực chiến: khi sự cố nằm ở phần WAN, việc xử lý phụ thuộc SLA với nhà cung cấp — sysadmin không tự khắc phục hoàn toàn được, chỉ có thể mở ticket và cung cấp bằng chứng (traceroute, thời điểm mất kết nối) để rút ngắn thời gian ISP xử lý.

## 4. Mô hình Cloud — IaaS, PaaS, SaaS

Theo NIST SP 800-145, ba mô hình dịch vụ cloud khác nhau ở **ai quản lý layer nào**:

```
              On-Premise      IaaS            PaaS            SaaS
              ------------    ------------    ------------    ------------
Ứng dụng      [Bạn]           [Bạn]           [Bạn]           [Nhà CC]
Dữ liệu       [Bạn]           [Bạn]           [Bạn]           [Bạn]
Runtime       [Bạn]           [Bạn]           [Nhà CC]        [Nhà CC]
OS            [Bạn]           [Bạn]           [Nhà CC]        [Nhà CC]
Ảo hóa        [Bạn]           [Nhà CC]        [Nhà CC]        [Nhà CC]
Server        [Bạn]           [Nhà CC]        [Nhà CC]        [Nhà CC]
Storage       [Bạn]           [Nhà CC]        [Nhà CC]        [Nhà CC]
Network       [Bạn]           [Nhà CC]        [Nhà CC]        [Nhà CC]
```

| Mô hình | Nhà cung cấp lo | Bạn vẫn phải lo | Ví dụ |
|---|---|---|---|
| **IaaS** | Server, storage, network, ảo hóa | OS, patch, runtime, ứng dụng, dữ liệu, quyền truy cập | AWS EC2, Azure VM |
| **PaaS** | + OS, runtime | Ứng dụng, dữ liệu, quyền truy cập | Azure App Service, Google App Engine |
| **SaaS** | + Ứng dụng | Dữ liệu, quyền truy cập người dùng | Microsoft 365, Google Workspace |

> [!question] Dùng SaaS rồi thì hết trách nhiệm bảo mật?
> Sai. Đây là ngộ nhận phổ biến nhất khi mới học cloud. Mô hình **shared responsibility**: dù ở tầng nào, **dữ liệu và việc quản lý danh tính/quyền truy cập (identity & access) luôn thuộc về doanh nghiệp**. SaaS chỉ giảm gánh nặng hạ tầng, không giảm trách nhiệm cấu hình phân quyền, MFA, chính sách chia sẻ file.

> [!example] Case study — Capital One 2019 (rò rỉ 100+ triệu hồ sơ khách hàng)
> Ngày 22-23/03/2019, kẻ tấn công khai thác một Web Application Firewall bị cấu hình sai trên hạ tầng AWS (IaaS) của Capital One bằng kỹ thuật SSRF, lấy được credential tạm thời của một IAM Role gắn cho EC2 instance. Role này bị cấp quyền quá rộng — đọc được **toàn bộ** S3 bucket chứa dữ liệu nhạy cảm và giải mã được dữ liệu — dẫn đến rò rỉ hơn 100.000 số an sinh xã hội cùng hàng triệu hồ sơ mở thẻ tín dụng (2005-2019), tổng cộng khoảng 30GB dữ liệu. Capital One bị phạt 80 triệu USD và bồi thường 190 triệu USD cho khách hàng. Đây là minh chứng trực tiếp cho mục "shared responsibility" ở trên: AWS (nhà cung cấp IaaS) chịu trách nhiệm hạ tầng vật lý và nền tảng ảo hóa, nhưng **việc cấu hình firewall, phân quyền IAM, và bảo vệ dữ liệu trong S3 vẫn là trách nhiệm của khách hàng** — không phải của AWS. Nguồn: [AppSecEngineer](https://www.appsecengineer.com/blog/aws-shared-responsibility-model-capital-one-breach-case-study), [Dark Reading](https://www.darkreading.com/cyberattacks-data-breaches/capital-one-attacker-exploited-misconfigured-aws-databases).

## 5. Kiến trúc mạng doanh nghiệp thực tế

### 5.1 Mô hình SOHO — điểm khởi đầu

```
                         Internet
                             |
                        +---------+
                        | Router  |
                        +---------+
                             |
                        +---------+
                        | Firewall|
                        +---------+
                             |
                        +---------+
                        | Switch  |   <- Star topology
                        +---------+
                        /    |    \
                    PC1     PC2    Server
```

**SOHO (Small Office/Home Office)**: gộp router + firewall + switch + Wi-Fi AP vào một thiết bị all-in-one. Phù hợp văn phòng dưới 20-30 máy, không có yêu cầu redundancy hay phân vùng bảo mật phức tạp.

### 5.2 Vì sao doanh nghiệp không dùng 1 switch phẳng cho cả tòa nhà

Mô hình SOHO không scale được khi số lượng thiết bị và số phòng ban tăng lên, vì 3 lý do thực chiến:

- **Miền broadcast quá lớn**: càng nhiều thiết bị trên cùng 1 switch/1 mạng phẳng, traffic broadcast (ARP, DHCP discover...) càng chiếm băng thông, làm chậm cả mạng dù không ai đang truyền dữ liệu thật.
- **Không phân vùng được theo phòng ban/mức tin cậy**: kế toán, HR, khách vãng lai (guest Wi-Fi) không thể tách biệt nhau nếu tất cả cùng nằm trên 1 switch không có VLAN.
- **Không có điểm dự phòng**: một switch trung tâm duy nhất là single point of failure cho toàn bộ tòa nhà — nếu hỏng, mọi phòng ban đều mất mạng cùng lúc.

### 5.3 Mô hình phân lớp: Two-Tier và Three-Tier

Doanh nghiệp giải quyết vấn đề trên bằng cách **phân lớp thiết bị theo chức năng** thay vì dùng 1 lớp switch phẳng. Hai mô hình phổ biến (nội dung cấu hình chi tiết ở Bài 4 và Bài 12 - Inter-VLAN Routing):

- **Two-Tier (Collapsed Core)**: gộp lớp Core và Distribution làm một. Phù hợp doanh nghiệp vừa, một tòa nhà vài tầng.
- **Three-Tier**: tách riêng 3 lớp Core – Distribution – Access. Phù hợp doanh nghiệp lớn, nhiều tòa nhà/campus.

```
                               Internet
                                  |
                             +---------+
                             |  Router |
                             +---------+
                                  |
                             +---------+
                             | Firewall|
                             +---------+
                                  |
                   +------------------------------+
                   |          CORE LAYER          |   <- backbone tốc độ cao,
                   |  (2 switch core, redundant)  |      không xử lý policy chi tiết
                   +------------------------------+
                                  |
          +---------------------+    +---------------------+
          |  DISTRIBUTION LAYER |    |  DISTRIBUTION LAYER |   <- ranh giới routing giữa
          |   (Switch Layer 3)  |    |   (Switch Layer 3)  |      VLAN, áp policy/ACL
          +---------------------+    +---------------------+
                                  |
           +---------+ +---------+ +---------+ +---------+
           |  ACCESS | |  ACCESS | |  ACCESS | |  ACCESS |         <- switch tầng, nơi
           |  Tầng 1 | |  Tầng 2 | | Kế toán | |  Server |            PC/AP cắm trực tiếp
           +---------+ +---------+ +---------+ +---------+
               PC, AP     PC, AP   PC (VLAN)    Server
```

| Lớp | Chức năng chính | Thiết bị điển hình |
|---|---|---|
| **Access** | Nơi end device kết nối trực tiếp (PC, AP, máy in) | Switch Layer 2 giá rẻ, nhiều cổng |
| **Distribution** | Ranh giới định tuyến giữa các VLAN, nơi áp policy/ACL/QoS | Switch Layer 3 |
| **Core** | Chuyển tiếp traffic tốc độ cao giữa các khối Distribution, không xử lý policy chi tiết để tránh làm chậm backbone | Switch Core hiệu năng cao, luôn có 2 thiết bị dự phòng nhau |

### 5.4 Case study nhỏ

Một công ty 3 tầng, ~200 nhân viên (kế toán, kinh doanh, kỹ thuật) triển khai theo Two-Tier: mỗi tầng có 1-2 switch Access nối PC/AP, tất cả switch Access uplink về 2 switch Distribution/Core đặt song song ở phòng máy chủ (redundant bằng EtherChannel — Bài 10). Mỗi phòng ban nằm trên 1 VLAN riêng (Bài 8) dù ngồi chung tầng, đảm bảo traffic broadcast của kế toán không ảnh hưởng đến kỹ thuật, và có thể áp policy truy cập khác nhau cho từng phòng ban ngay tại lớp Distribution.

- **SOHO vs Enterprise**: SOHO gộp mọi chức năng vào 1 thiết bị, chấp nhận không có redundancy vì quy mô nhỏ, chi phí downtime thấp. Enterprise tách lớp để mở rộng dễ, phân quyền quản trị theo lớp (đội network chỉ quản Core/Distribution, đội helpdesk quản Access), và luôn có dự phòng ở lớp Core vì đây là điểm ảnh hưởng toàn hệ thống nếu hỏng.

### 5.5 Bức tranh đầy đủ hơn — DMZ, Wireless, WAN Edge, Management network

Mục 5.3 mới chỉ vẽ phần **Enterprise Campus** (mạng LAN nội bộ trong một khuôn viên). Theo Cisco Enterprise Architecture Model, một kiến trúc doanh nghiệp thực tế còn có thêm các khối sau mà sơ đồ ở trên đang thiếu:

**Nhánh Internet Edge (Router → Firewall → DMZ):**

```
     Internet
        |
   +---------+
   |  Router |
   +---------+
        |
   +---------+
   | Firewall|
   +---------+
        |
+----------------+
|      DMZ       |
|   (Web/Mail    |
| server public) |
+----------------+
```

**Nhánh Core kết nối WAN Edge (sang chi nhánh) và Wireless LAN Controller** — Core Layer ở đây chính là Core Layer đã vẽ trong mô hình Two-Tier/Three-Tier ở mục 5.3, chỉ vẽ tách riêng để không làm rối sơ đồ:

```
(Core Layer — tiếp nối từ mô hình mục 5.3 ở trên)
              +--------------+
              |  CORE LAYER  |
              +--------------+
                      |
+-----------------+    +--------------------+
| WAN Edge Router |    |    Wireless LAN    |
+-----------------+    |  Controller (WLC)  |
                       | quản lý tập trung  |
                       |  mọi Access Point  |
                       +--------------------+
         |
+------------------+
|   Chi nhánh 2    |
|  (WAN: MPLS/VPN  |
|  site-to-site)   |
+------------------+
```

- **DMZ (Demilitarized Zone)**: vùng đặt server cần truy cập từ Internet (web, mail) — tách khỏi LAN nội bộ. Nếu server trong DMZ bị tấn công, kẻ tấn công không nhảy thẳng được vào mạng nội bộ (cùng nguyên tắc least-privilege segmentation dùng trong thiết kế firewall).

> [!example] Case study — Target 2013 (lộ dữ liệu ~110 triệu khách hàng vì thiếu segmentation)
> Ngày 15/11/2013, kẻ tấn công dùng credential đánh cắp từ Fazio Mechanical Services — nhà thầu HVAC được Target cấp quyền truy cập từ xa để giám sát nhiệt độ/năng lượng tại các cửa hàng — để xâm nhập vào Vendor Portal. Lỗi thiết kế mấu chốt: **cổng vendor này nằm chung mạng với hệ thống nhạy cảm**, không được tách vùng (segmentation) như một DMZ đúng nghĩa, nên kẻ tấn công di chuyển ngang (lateral movement) từ hệ thống HVAC sang thẳng hệ thống POS (điểm bán hàng). Kết quả: từ 27/11 đến 15/12/2013, khoảng 40 triệu số thẻ thanh toán và dữ liệu cá nhân của 70 triệu khách hàng bị đánh cắp (tổng ~110 triệu người bị ảnh hưởng), Target thiệt hại hơn 200 triệu USD. Đây chính là lý do nguyên tắc "DMZ/vendor access không được đi thẳng vào mạng nội bộ" ở trên không phải lý thuyết suông — thiếu nó là nguyên nhân trực tiếp của một trong các vụ rò rỉ dữ liệu lớn nhất lịch sử bán lẻ. Nguồn: [Krebs on Security](https://krebsonsecurity.com/2014/02/target-hackers-broke-in-via-hvac-company/), [Computerworld](https://www.computerworld.com/article/1517305/target-breach-happened-because-of-a-basic-network-segmentation-error.html).
- **Wireless LAN Controller (WLC)**: quản lý tập trung toàn bộ Access Point trong tòa nhà thay vì cấu hình tay từng AP — chi tiết cấu hình ở Bài 24-25.
- **WAN Edge**: router biên kết nối ra ngoài — lên Internet qua ISP, hoặc sang chi nhánh khác qua MPLS thuê riêng/VPN site-to-site dựng trên nền Internet (Bài 22).
- **Management network** (thường không vẽ trên sơ đồ tổng quan nhưng luôn tồn tại thực tế): VLAN/mạng riêng chỉ dùng để quản trị thiết bị (SSH vào switch/router/firewall), tách khỏi VLAN người dùng để tránh user thường chạm được vào giao diện quản trị hạ tầng.

Cập nhật case study ở mục 5.4: nếu công ty đó mở thêm chi nhánh ở tỉnh khác, hai văn phòng sẽ nối với nhau qua WAN (MPLS hoặc VPN site-to-site), và công ty đặt một dải server ở vùng DMZ cho website truy cập được từ Internet mà không ảnh hưởng đến mạng nội bộ nơi nhân viên đang làm việc.

## 6. Ngộ nhận thường gặp khi mới tiếp cận

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| LAN = mạng Wi-Fi trong nhà | LAN là khái niệm phạm vi, không gắn với công nghệ truyền dẫn cụ thể | LAN = mạng cục bộ trong khu vực giới hạn, có dây hay không dây đều được |
| Ring topology đã biến mất hoàn toàn | Vẫn tồn tại ở tầng backbone/WAN một số nhà mạng nhờ khả năng dự phòng vòng kép | Ring hiếm gặp ở LAN doanh nghiệp, không phải đã biến mất khỏi hạ tầng mạng nói chung |
| MAN và WAN là một | Cả hai đều "lớn hơn LAN" nhưng khác phạm vi địa lý và đơn vị sở hữu | MAN giới hạn trong 1 thành phố/khu vực; WAN vượt ra ngoài, thường đi thuê ISP |

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer (không cần cấu hình IOS ở bài này, mục tiêu là quan sát topology).

1. Dựng 4 PC theo mô hình Bus (mô phỏng qua cáp nối chuỗi).
2. Dựng lại 4 PC đó theo mô hình Star, dùng 1 Switch trung tâm.
3. Rút cáp 1 PC bất kỳ ở cả hai mô hình, quan sát 3 PC còn lại có ping thông nhau không.
4. Ghi nhận sự khác biệt về fault tolerance giữa hai mô hình — đây chính là minh chứng trực quan cho mục 2.

**Kết quả kỳ vọng**: ở Star, 3 PC còn lại vẫn ping thông bình thường sau khi rút cáp 1 PC; ở Bus, việc "đứt" đoạn cáp trục ảnh hưởng đến các PC nằm sau điểm đứt.

## 8. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao Star topology được dùng phổ biến hơn Bus và Ring trong LAN doanh nghiệp?**
→ Vì khả năng cô lập sự cố: một thiết bị hoặc đoạn cáp hỏng chỉ ảnh hưởng đến chính nó, không kéo sập toàn mạng như Bus, và không cần cả vòng như Ring. Đánh đổi là switch trung tâm trở thành single point of failure, cần thiết kế redundancy ở các bài sau.

**2. Phân biệt LAN, MAN, WAN — không chỉ theo phạm vi mà theo trách nhiệm vận hành.**
→ LAN nằm trong tòa nhà/văn phòng, doanh nghiệp tự sở hữu và toàn quyền cấu hình. MAN trong phạm vi thành phố. WAN liên tỉnh/quốc gia, thường đi thuê từ ISP — khi lỗi ở WAN, xử lý phụ thuộc SLA nhà cung cấp chứ không tự khắc phục hoàn toàn.

**3. IaaS, PaaS, SaaS khác nhau ở điểm nào? Dùng SaaS thì hết trách nhiệm bảo mật đúng không?**
→ Khác nhau ở việc nhà cung cấp quản lý đến layer nào (hạ tầng/OS/ứng dụng). Sai — theo mô hình shared responsibility, dữ liệu và quản lý danh tính/quyền truy cập luôn thuộc về doanh nghiệp dù ở mô hình nào.

**4. Một PC trong mạng Star bị mất kết nối, các PC khác vẫn hoạt động bình thường. Bước đầu tiên cần kiểm tra là gì?**
→ Kiểm tra đúng cổng switch nối với PC đó (cáp, trạng thái đèn tín hiệu, cấu hình port) trước khi nghi ngờ toàn mạng — đặc điểm cô lập sự cố của Star cho phép khoanh vùng ngay từ đầu.

## 9. Tài liệu tham khảo

- [Cisco CCNA 200-301 – Bus, Ring, Star & Mesh Topologies](https://www.certificationkits.com/ccna-200-301-topic-articles/cisco-ccna-200-301-physical-networking-concepts-layer-1/cisco-ccna-200-301-bus-ring-star-mesh-topologies/)
- [Generic topologies - Ring, Star, Bus | NetworkAcademy.IO](https://www.networkacademy.io/ccna/network-fundamentals/generic-topologies-ring-star-bus)
- [NIST Special Publication 500-322 – Evaluation of Cloud Computing](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.500-322.pdf)
- [NIST definition for SaaS, PaaS, IaaS](https://cloudinfosec.wordpress.com/2013/05/04/nist-definition-for-saas-paas-iaas/)
- [Cisco Enterprise Campus 3.0 Architecture — Overview and Framework](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Campus/campover.html)
- [Engineering at Meta — More details about the October 4 outage](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)
- [AWS Shared Responsibility Model: Capital One Breach Case Study](https://www.appsecengineer.com/blog/aws-shared-responsibility-model-capital-one-breach-case-study)
- [Krebs on Security — Target Hackers Broke in Via HVAC Company](https://krebsonsecurity.com/2014/02/target-hackers-broke-in-via-hvac-company/)

---
**Liên kết**: [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2 - Mô hình OSI, TCP/IP →]]
