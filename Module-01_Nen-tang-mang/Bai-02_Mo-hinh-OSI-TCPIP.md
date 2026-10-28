---
title: Bài 2 - Mô hình OSI, TCP/IP
type: Foundation
difficulty: Beginner
lab_required: true
exam: "Sysadmin Junior — ngôn ngữ tư duy bắt buộc để đọc hiểu mọi tài liệu troubleshoot mạng"
tags: [networking, fundamentals, osi, tcp-ip, mac-ip, arp, ports, sysadmin]
aliases: [OSI Model, TCP/IP Model, Mo hinh OSI TCPIP]
related: ["[[Bai-01_Gioi-thieu-mang-may-tinh]]", "[[Bai-03_Phuong-tien-truyen-dan-mang]]"]
last_review: 2026-07-21
---

# Bài 2: Mô hình OSI, TCP/IP

> [!important] Tại sao cần file này
> Khi một switch/network engineer nói "đây là vấn đề Layer 2" hay "chắc Layer 3 rồi", họ đang dùng chung một ngôn ngữ chuẩn hóa để khoanh vùng sự cố cực nhanh mà không cần giải thích dài dòng. Không hiểu mô hình OSI/TCP-IP, sysadmin sẽ không đọc hiểu được bất kỳ tài liệu troubleshoot, log thiết bị, hay câu hỏi phỏng vấn nào trong nghề — vì gần như toàn bộ thuật ngữ ngành (frame, packet, segment, Layer 2 loop, Layer 3 routing...) đều bắt nguồn từ đây.

## 1. Vì sao cần một mô hình chuẩn hóa

Trước khi có mô hình chuẩn, mỗi hãng thiết bị xây dựng giao thức mạng theo cách riêng — thiết bị hãng A không nói chuyện được với hãng B. Tổ chức ISO (International Organization for Standardization) xây dựng **mô hình OSI (Open Systems Interconnection)** làm khung tham chiếu chung: chia quá trình truyền dữ liệu thành 7 lớp độc lập, mỗi lớp chỉ cần biết cách "nói chuyện" với lớp ngay trên và ngay dưới nó, không quan tâm chi tiết bên trong các lớp khác.

Điểm cần phân biệt rõ ngay từ đầu: **OSI là mô hình tham chiếu lý thuyết** (không ai cài đặt y hệt 7 lớp), còn **TCP/IP là bộ giao thức thực tế đang chạy Internet** (4 lớp, ra đời từ dự án ARPANET trước cả khi OSI hoàn thiện). Sysadmin dùng song song cả hai: từ vựng "Layer 1-7" để tư duy/troubleshoot, nhưng thiết bị thực tế đóng gói dữ liệu theo TCP/IP.

## 2. Mô hình OSI — 7 lớp

| Lớp | Tên | PDU (đơn vị dữ liệu) | Chức năng chính | Giao thức/thiết bị điển hình |
|---|---|---|---|---|
| 7 | Application | Data | Giao diện để ứng dụng người dùng truy cập dịch vụ mạng | HTTP, HTTPS, FTP, DNS, SMTP |
| 6 | Presentation | Data | Định dạng, mã hóa, nén dữ liệu — đảm bảo 2 đầu "hiểu" cùng định dạng | TLS/SSL, JPEG, ASCII |
| 5 | Session | Data | Thiết lập, duy trì, kết thúc phiên làm việc giữa 2 ứng dụng | NetBIOS, RPC, SIP |
| 4 | Transport | Segment (TCP) / Datagram (UDP) | Phân đoạn dữ liệu, đảm bảo tin cậy hoặc tốc độ, quản lý port | TCP, UDP |
| 3 | Network | Packet | Đánh địa chỉ logic (IP), định tuyến giữa các mạng | IP, ICMP, Router |
| 2 | Data Link | Frame | Đánh địa chỉ vật lý (MAC), đóng khung, kiểm tra lỗi cục bộ | Ethernet, Switch, MAC address |
| 1 | Physical | Bit | Truyền tín hiệu điện/quang/sóng qua môi trường vật lý | Cáp UTP, cáp quang, Hub |

**Quá trình đóng gói (encapsulation)** — mỗi lớp khi truyền xuống lớp dưới sẽ thêm một header (và đôi khi trailer) của riêng mình, bọc quanh dữ liệu của lớp trên:

```
Data (Lớp 7-5: Application/Presentation/Session)
        |
        v  Lớp 4 thêm TCP/UDP Header
+--------+-----------+
| TCP Hdr|    Data   |   <- SEGMENT
+--------+-----------+
        |
        v  Lớp 3 thêm IP Header
+--------+--------+-----------+
| IP Hdr | TCP Hdr|    Data   |   <- PACKET
+--------+--------+-----------+
        |
        v  Lớp 2 thêm Ethernet Header + FCS Trailer
+---------+--------+--------+-----------+-----+
| Eth Hdr | IP Hdr | TCP Hdr|    Data   | FCS |   <- FRAME
+---------+--------+--------+-----------+-----+
        |
        v  Lớp 1 chuyển thành tín hiệu
   0101101010110...   <- BIT
```

Ở đầu nhận, quá trình diễn ra ngược lại (**decapsulation**) — mỗi lớp bóc header của chính mình rồi đẩy phần còn lại lên lớp trên. Đây chính là cơ chế giải thích vì sao khi troubleshoot, việc đầu tiên luôn là "đi từ dưới lên" (Physical → Data Link → Network...) — vì lớp dưới hỏng thì lớp trên không có gì để bóc ra cả.

### 2.1 Lớp 1 — Physical

Nhiệm vụ duy nhất: chuyển đổi bit (0/1) thành tín hiệu vật lý thật sự đi qua môi trường truyền dẫn — điện áp trên cáp đồng, ánh sáng trong cáp quang, hoặc sóng vô tuyến. Lớp này không quan tâm dữ liệu bên trong là gì, chỉ quan tâm chuẩn cáp/connector, tốc độ (10/100/1000 Mbps...), và encoding tín hiệu có đúng chuẩn hay không.

- **Thiết bị/thành phần điển hình**: cáp UTP, cáp quang, Hub, transceiver SFP, cổng RJ45.
- **Hậu quả khi lỗi**: cáp đứt/lỏng, cổng quang bẩn, sai chuẩn tốc độ-duplex giữa 2 đầu → mất kết nối hoàn toàn hoặc kết nối chập chờn (interface flapping). Đây luôn là lớp đầu tiên cần loại trừ khi troubleshoot: đèn tín hiệu trên cổng switch có sáng không, cáp có cắm chặt không — vì mọi lớp phía trên đều vô nghĩa nếu Lớp 1 không hoạt động.

### 2.2 Lớp 2 — Data Link

Đóng khung dữ liệu thành frame, gắn địa chỉ MAC nguồn/đích, và kiểm tra lỗi truyền cục bộ bằng trường FCS (Frame Check Sequence — một mã CRC): nếu bit bị sai lệch trong quá trình truyền, thiết bị nhận tính lại CRC không khớp thì **drop thẳng frame đó**, không cố sửa lỗi.

- **Thiết bị/thành phần điển hình**: Switch (học và lưu bảng MAC address table theo từng port), Ethernet, VLAN tag (802.1Q).
- **Hậu quả khi lỗi**: switching loop gây broadcast storm (Bài 10 — STP tồn tại để ngăn việc này), MAC flapping (một MAC học được luân phiên trên nhiều port — dấu hiệu của loop hoặc thiết bị giả mạo), VLAN mismatch giữa 2 đầu trunk. Đặc điểm dễ nhận biết: đèn tín hiệu Lớp 1 vẫn sáng bình thường nhưng thiết bị vẫn không liên lạc được — chứng tỏ sự cố đã lên đến Lớp 2.

### 2.3 Lớp 3 — Network

Đánh địa chỉ logic (IP) và quyết định đường đi (định tuyến) cho gói tin giữa các mạng khác nhau. Router nhận packet, tra bảng định tuyến (routing table) để tìm next-hop dựa trên IP đích, sau đó bóc frame Lớp 2 cũ và đóng gói lại frame Lớp 2 mới cho chặng tiếp theo — riêng IP header gần như giữ nguyên xuyên suốt hành trình (chỉ giảm TTL mỗi lần qua 1 router, trừ khi có NAT).

- **Thiết bị/thành phần điển hình**: Router, giao thức IP, ICMP, các giao thức định tuyến OSPF/EIGRP (Bài 14-15).
- **Hậu quả khi lỗi**: sai default gateway, thiếu route trong bảng định tuyến, TTL expired do routing loop → triệu chứng đặc trưng: máy vẫn ping được các thiết bị **cùng subnet** (vì đó là việc của Lớp 2) nhưng không ra được mạng khác/Internet.

### 2.4 Lớp 4 — Transport

Chia nhỏ dữ liệu ứng dụng thành segment (TCP) hoặc datagram (UDP), và quan trọng nhất: gắn **số port** để phân biệt nhiều ứng dụng đang chạy đồng thời trên cùng một host (ví dụ máy chủ vừa chạy web server cổng 80 vừa chạy SSH cổng 22). TCP theo dõi trạng thái bằng sequence number/ACK để đảm bảo dữ liệu đến đúng thứ tự và truyền lại phần bị mất; UDP không theo dõi trạng thái, gửi xong là xong.

- **Thiết bị/thành phần điển hình**: TCP, UDP, khái niệm port number (mục 5).
- **Hậu quả khi lỗi**: firewall/ACL chặn nhầm port khiến ứng dụng "treo" dù `ping` (hoạt động ở Lớp 3) vẫn thông bình thường. Đây là lý do ping thành công **không đồng nghĩa** với việc ứng dụng chạy được — phải kiểm tra đúng port bằng công cụ như `telnet <ip> <port>` hoặc `nc -zv <ip> <port>`.

### 2.5 Lớp 5 — Session

Thiết lập, duy trì, đồng bộ và kết thúc "phiên" trao đổi giữa 2 ứng dụng ở hai đầu — cho phép checkpoint để nối lại phiên nếu bị gián đoạn giữa chừng, thay vì phải bắt đầu lại từ đầu. Lớp này quản lý dialog control: bên nào được "nói" trước, cơ chế đồng bộ khi có sự cố tạm thời.

- **Thiết bị/thành phần điển hình**: NetBIOS session, RPC (Remote Procedure Call), cơ chế phiên đăng nhập (login session).
- **Hậu quả khi lỗi/liên hệ thực tế**: session timeout, mất phiên làm việc khi VPN hoặc remote desktop rớt giữa chừng dù bản thân kết nối mạng bên dưới (Lớp 1-4) vẫn còn — ứng dụng phải đăng nhập lại vì "phiên" logic đã bị hủy chứ không phải vì mất mạng vật lý.

### 2.6 Lớp 6 — Presentation

Chuyển đổi định dạng dữ liệu để hai hệ thống khác nhau (khác hệ điều hành, khác chuẩn ký tự, khác cách biểu diễn dữ liệu) hiểu được nhau — bao gồm mã hóa/giải mã và nén/giải nén. Đây là lớp mà quá trình bắt tay TLS/SSL (mã hóa dữ liệu trước khi xuống Lớp 4) thực chất diễn ra.

- **Thiết bị/thành phần điển hình**: TLS/SSL, định dạng file (JPEG, PNG), chuẩn ký tự (ASCII, Unicode).
- **Hậu quả khi lỗi**: certificate hết hạn hoặc cipher suite không khớp giữa 2 đầu trong lúc bắt tay TLS → trình duyệt báo lỗi bảo mật, dù bản thân kết nối TCP (Lớp 4) đã thiết lập thành công (`telnet` vào cổng 443 vẫn connect được).

### 2.7 Lớp 7 — Application

Giao diện trực tiếp để phần mềm người dùng gọi đến dịch vụ mạng — cần phân biệt rõ: **trình duyệt không phải là Lớp 7**, mà giao thức nó sử dụng (HTTP/HTTPS) mới là Lớp 7. Lớp này định nghĩa cú pháp yêu cầu/phản hồi cụ thể cho từng dịch vụ (ví dụ lệnh GET/POST của HTTP, các lệnh của giao thức SMTP khi gửi mail).

- **Thiết bị/thành phần điển hình**: HTTP/HTTPS, DNS, SMTP, FTP.
- **Hậu quả khi lỗi**: server trả về lỗi 404/500, hoặc ứng dụng crash — dù toàn bộ 6 lớp bên dưới hoàn toàn bình thường (mạng thông, TCP connect được). Đây là lý do khi người dùng báo "web không vào được", sysadmin vẫn phải kiểm tra riêng tình trạng ứng dụng/server, không thể kết luận chỉ dựa vào việc mạng đã thông.

## 3. Mô hình TCP/IP — 4 lớp và ánh xạ sang OSI

| TCP/IP (4 lớp) | Tương ứng OSI | Ghi chú |
|---|---|---|
| Application | Application + Presentation + Session (Lớp 7-5) | TCP/IP gộp 3 lớp trên cùng thành 1 |
| Transport | Transport (Lớp 4) | Giữ nguyên như OSI |
| Internet | Network (Lớp 3) | Giữ nguyên chức năng, chỉ khác tên gọi |
| Network Access (Link) | Data Link + Physical (Lớp 2-1) | TCP/IP gộp 2 lớp dưới cùng thành 1 |

> [!question] Vậy học OSI hay TCP/IP mới đúng?
> Cả hai — không phải chọn 1. TCP/IP là những gì thiết bị thực sự đóng gói và truyền đi (thứ bạn thấy khi bật Wireshark). OSI là ngôn ngữ tư duy chuẩn ngành để mô tả và khoanh vùng sự cố ("Layer 2 down", "định tuyến Layer 3 sai") — kể cả khi công nghệ thực tế chạy TCP/IP 4 lớp, dân mạng vẫn nói theo 7 lớp OSI vì nó chi tiết hơn, đặc biệt tách riêng được Physical với Data Link và tách Session/Presentation ra khỏi Application.

## 4. Địa chỉ MAC vs Địa chỉ IP

| | MAC Address | IP Address |
|---|---|---|
| Thuộc lớp | 2 — Data Link | 3 — Network |
| Bản chất | Vật lý, ghi cứng vào NIC bởi nhà sản xuất (burned-in) | Logic, gán bởi admin hoặc DHCP |
| Cấu trúc | 48-bit, dạng hex, VD: `00:1A:2B:3C:4D:5E` | 32-bit (IPv4), dạng thập phân, VD: `192.168.1.10` |
| Phạm vi hoạt động | Chỉ có ý nghĩa trong cùng 1 segment/broadcast domain | Định tuyến được xuyên nhiều mạng khác nhau (routable) |
| Có đổi được không | Cố định trên phần cứng (dù có thể giả mạo bằng MAC spoofing) | Gán linh hoạt, đổi được theo thiết kế mạng |

> [!example] Case study — IANA cạn kiệt IPv4 pool (03/02/2011)
> Ngày 03/02/2011, IANA cấp phát 2 khối địa chỉ IPv4 cuối cùng cho APNIC, kích hoạt chính sách chia đều phần còn lại cho 5 tổ chức khu vực (RIR) — đánh dấu **kho địa chỉ IPv4 toàn cầu chính thức cạn kiệt**. Nguyên nhân gốc rễ: không gian IPv4 32-bit chỉ có khoảng 4,3 tỷ địa chỉ, được thiết kế từ những năm 1980 khi không ai hình dung Internet sẽ phổ cập đến từng chiếc điện thoại, TV, camera IP. Đây chính là động lực trực tiếp khiến IPv6 (128-bit, không gian địa chỉ gần như vô hạn) ra đời và dần được triển khai — nội dung chi tiết ở Bài 6. Nguồn: [ARIN](https://www.arin.net/vault/announcements/20110203/), [NRO](https://www.nro.net/ipv4-free-pool-depleted/).

## 5. TCP vs UDP, Port, ICMP

| | TCP | UDP |
|---|---|---|
| Kết nối | Connection-oriented — bắt tay 3 bước trước khi gửi dữ liệu | Connectionless — gửi thẳng, không bắt tay |
| Độ tin cậy | Đảm bảo, có xác nhận (ACK), tự động truyền lại gói mất | Best-effort, không đảm bảo đến nơi |
| Tốc độ | Chậm hơn do overhead kiểm soát | Nhanh hơn, gần như không overhead |
| Ứng dụng điển hình | Web (HTTP/HTTPS), email, truyền file | Streaming, VoIP, DNS query, DHCP |

**3-way handshake của TCP** (bắt buộc trước khi truyền dữ liệu thật):

```
Client                          Server
  |------------ SYN ----------->|
  |<--------- SYN/ACK ----------|
  |------------ ACK ----------->|
        (kết nối ESTABLISHED, bắt đầu truyền data)
```

**Một số port phổ biến cần thuộc lòng:**

| Port | Giao thức tầng 4 | Dịch vụ |
|---|---|---|
| 20/21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3389 | TCP | RDP |

**ICMP (Internet Control Message Protocol)** không dùng port và không thuộc TCP hay UDP — đây là giao thức độc lập ở Lớp 3, dùng cho mục đích chẩn đoán/báo lỗi mạng (`ping` dùng ICMP Echo Request/Reply, `traceroute` dùng ICMP Time Exceeded).

> [!example] Case study — Cuộc tấn công DDoS vào Dyn DNS (21/10/2016)
> Botnet Mirai — gồm khoảng 100.000 thiết bị IoT bị nhiễm mã độc (camera IP, router gia dụng...) — tấn công DDoS vào nhà cung cấp DNS Dyn với cường độ khoảng 1,2 Tbps. Kỹ thuật chính: **TCP SYN flood** nhắm vào port 53 (port mặc định của DNS) tại các data center của Dyn, kết hợp lưu lượng UDP giả mạo cũng qua port 53. Hậu quả: hàng loạt dịch vụ lớn như Twitter, Reddit, CNN, Pinterest không truy cập được trong gần 9 giờ vì DNS — dịch vụ phân giải tên miền — bị nghẽn. Case study này minh họa trực tiếp 2 khái niệm vừa học: TCP SYN flood khai thác đúng cơ chế 3-way handshake (gửi SYN liên tục nhưng không hoàn tất bắt tay, làm cạn tài nguyên server), còn việc tấn công tập trung vào **port 53** cho thấy vì sao hiểu rõ port/dịch vụ tương ứng lại quan trọng khi đọc log hoặc cấu hình ACL. Nguồn: [ThousandEyes](https://www.thousandeyes.com/blog/dyn-dns-ddos-attack), [DarkReading](https://www.darkreading.com/cyberattacks-data-breaches/ddos-on-dyn-used-malicious-tcp-udp-traffic).

## 6. Collision Domain và Broadcast Domain

| | Collision Domain | Broadcast Domain |
|---|---|---|
| Định nghĩa | Vùng mạng mà nếu 2 thiết bị truyền tín hiệu cùng lúc sẽ xảy ra đụng độ (collision) | Vùng mạng mà một khung tin broadcast gửi ra sẽ được mọi thiết bị trong đó nhận |
| Bị chia bởi | Switch (mỗi port = 1 collision domain riêng), Router | Chỉ Router hoặc VLAN — **switch KHÔNG chia được** |
| Vấn đề nếu quá lớn | Đụng độ tín hiệu liên tục, giảm hiệu năng (chủ yếu ý nghĩa lịch sử với Hub/half-duplex) | Broadcast storm — traffic broadcast tự nhân bản chiếm hết băng thông, có thể làm sập cả mạng |

```
HUB (1 collision domain chung)         SWITCH (mỗi port 1 collision domain)
     PC1                                    PC1
      \                                      |
       Hub---PC2   <- mọi cổng cùng          Switch---PC2   <- mỗi cổng độc lập,
      /       đụng độ chung 1 vùng          /              không đụng độ nhau
     PC3                                   PC3

  Cả 2 mô hình trên: nếu chỉ có switch/hub, KHÔNG có router,
  toàn bộ PC1/PC2/PC3 vẫn nằm chung 1 BROADCAST DOMAIN.
```

> [!question] Vì sao switch chia được collision domain nhưng không chia được broadcast domain?
> Vì switch vẫn chuyển tiếp (flood) frame broadcast ra **tất cả** các cổng còn lại (trừ cổng nhận vào) theo đúng thiết kế — đó là hành vi bắt buộc để đảm bảo mọi thiết bị trong mạng LAN đều nhận được broadcast (ví dụ ARP Request). Chỉ có Router hoặc việc chia VLAN (Bài 8) mới tạo ra ranh giới thật sự chặn broadcast lan sang phần mạng khác.

> [!example] Case study — Broadcast storm trong báo cáo NERC (lưới điện Bắc Mỹ)
> Báo cáo "Lessons Learned" của NERC (North American Electric Reliability Corporation) ghi nhận một sự cố tại trạm đang thi công: một switching loop (vòng lặp chuyển mạch) gây ra broadcast storm khiến các switch bị quá tải CPU gần như 100%, IP quản trị không truy cập được, gói tin mất tới gần 100%. Đội vận hành phải cô lập thủ công từng đoạn mạng, dò từng cổng cho đến khi tìm ra và tắt đúng cổng/thiết bị gây vòng lặp. Đây là ví dụ thực chiến cho việc **vì sao chỉ cần thiếu một cơ chế chống loop (STP — học ở Bài 10) là toàn bộ hạ tầng có thể sập trong vài giây**, và vì sao khi mạng "đứng hình toàn bộ" đột ngột, broadcast storm luôn là nghi vấn hàng đầu. Nguồn: [NERC Lessons Learned](https://www.nerc.com/globalassets/programs/event-analysis/lessons-learned/ll20181001_networking_packet_broadcast_storms.pdf).

## 7. ARP — Address Resolution Protocol

Trong cùng một mạng LAN, để gửi được frame ở Lớp 2, thiết bị **bắt buộc phải biết địa chỉ MAC** của bên nhận — dù đã biết địa chỉ IP. ARP là giao thức làm nhiệm vụ "dịch" từ IP sang MAC trong phạm vi cùng broadcast domain.

```
PC A (biết IP của PC B, chưa biết MAC) muốn gửi dữ liệu cho PC B:

1. PC A gửi ARP Request (BROADCAST tới FF:FF:FF:FF:FF:FF):
   "Ai đang giữ IP 192.168.1.5? Cho tôi biết địa chỉ MAC."

2. Mọi thiết bị trong broadcast domain đều nhận được,
   chỉ PC B (đúng chủ IP đó) mới trả lời.

3. PC B gửi ARP Reply (UNICAST, chỉ gửi lại đúng PC A):
   "Tôi là 192.168.1.5, MAC của tôi là AA:BB:CC:DD:EE:FF"

4. PC A lưu kết quả vào ARP cache (bộ nhớ tạm),
   từ đó gửi thẳng frame tới đúng MAC mà không cần hỏi lại.
```

Vì ARP Reply không có cơ chế xác thực, một thiết bị bất kỳ trong cùng LAN về lý thuyết có thể trả lời giả mạo (ARP spoofing/poisoning), khiến 2 thiết bị khác gửi nhầm traffic qua máy của kẻ tấn công trước khi đến đích thật. Đây chính là lý do các switch doanh nghiệp có tính năng Dynamic ARP Inspection (DAI — Bài 27) để kiểm tra tính hợp lệ của ARP Reply trước khi cho qua.

## 8. Lab thực hành

**Môi trường**: Cisco Packet Tracer hoặc Wireshark trên máy thật.

1. Dựng 2 PC nối qua 1 Switch, đặt IP cùng subnet.
2. Xóa ARP cache trên PC A (`arp -d *` trên Windows), sau đó `ping` sang PC B.
3. Bật capture (Simulation mode trên Packet Tracer, hoặc Wireshark trên máy thật) và quan sát: gói ARP Request/Reply xuất hiện **trước** gói ICMP đầu tiên.
4. Mở chi tiết gói ICMP vừa capture, xác định từng lớp header: Ethernet (MAC nguồn/đích) → IP (IP nguồn/đích) → ICMP (type/code) — đúng như trình tự encapsulation ở mục 2.

**Kết quả kỳ vọng**: nhìn thấy rõ ràng ARP diễn ra trước khi ping thành công lần đầu, và xác nhận được cấu trúc đóng gói nhiều lớp lồng nhau trong một gói tin thật, không chỉ trên lý thuyết.

## 9. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| IP address quan trọng hơn MAC address | Trong cùng 1 subnet, việc gửi frame ở Lớp 2 luôn dựa vào MAC — IP chỉ dùng để định tuyến xuyên mạng | Cả hai đều bắt buộc: IP để biết "đi đâu" (định tuyến), MAC để biết "gửi cho ai" trên đoạn mạng cục bộ cuối cùng |
| OSI và TCP/IP là 2 mô hình cạnh tranh, học 1 cái là đủ | Đây là 2 công cụ khác mục đích, không thay thế nhau | TCP/IP là thứ thiết bị thực sự chạy; OSI là ngôn ngữ troubleshoot chi tiết hơn — dùng song song |
| Switch chia được broadcast domain giống collision domain | Switch chỉ chia collision domain theo từng port, vẫn flood broadcast ra mọi port | Chỉ Router hoặc VLAN mới tạo ranh giới broadcast domain thật sự |
| ICMP là một loại của TCP hoặc UDP | ICMP là giao thức độc lập ở Lớp 3, không có khái niệm port như TCP/UDP ở Lớp 4 | ICMP dùng riêng cho chẩn đoán/báo lỗi (ping, traceroute), không mang dữ liệu ứng dụng |

## 10. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Encapsulation là gì? Trình bày thứ tự từ Lớp 7 xuống Lớp 1.**
→ Là quá trình mỗi lớp thêm header (và trailer ở Lớp 2) của riêng mình khi dữ liệu đi từ trên xuống: Data (App/Presentation/Session) → Segment (Transport, thêm TCP/UDP header) → Packet (Network, thêm IP header) → Frame (Data Link, thêm Ethernet header + FCS) → Bit (Physical).

**2. PDU của Transport, Network, Data Link, Physical lần lượt là gì?**
→ Transport: Segment (TCP) hoặc Datagram (UDP). Network: Packet. Data Link: Frame. Physical: Bit.

**3. Phân biệt collision domain và broadcast domain — switch chia được cái nào?**
→ Collision domain là vùng có thể xảy ra đụng độ tín hiệu, switch chia được (mỗi port riêng). Broadcast domain là vùng nhận chung frame broadcast, switch KHÔNG chia được — chỉ Router hoặc VLAN mới chia được.

**4. ARP hoạt động ra sao? Bước nào dùng broadcast, bước nào dùng unicast?**
→ ARP Request dùng broadcast (gửi tới FF:FF:FF:FF:FF:FF, mọi thiết bị trong broadcast domain nhận được) để hỏi "ai giữ IP này". ARP Reply dùng unicast, chỉ chủ nhân IP đó trả lời trực tiếp về máy hỏi, kèm địa chỉ MAC.

**5. TCP khác UDP ở điểm nào? Khi nào chọn UDP dù kém tin cậy hơn?**
→ TCP connection-oriented, có bắt tay 3 bước, đảm bảo tin cậy nhưng chậm hơn. UDP connectionless, best-effort, nhanh hơn. Chọn UDP khi ứng dụng ưu tiên tốc độ/độ trễ thấp hơn độ tin cậy tuyệt đối — ví dụ streaming, VoIP, DNS query (mất gói thì hỏi lại nhanh hơn là chờ TCP retransmit).

**6. Một gói tin đi qua 3 router trước khi đến đích. Địa chỉ MAC và IP nguồn/đích thay đổi ra sao ở mỗi chặng?**
→ Địa chỉ IP nguồn/đích giữ nguyên xuyên suốt hành trình (trừ khi có NAT). Địa chỉ MAC nguồn/đích **thay đổi ở mỗi chặng** — mỗi router bóc frame cũ, định tuyến theo IP đích, rồi đóng gói lại frame mới với MAC nguồn là chính router đó và MAC đích là thiết bị/router kế tiếp trên đường đi.

## 11. Tài liệu tham khảo

- [OSI Model Explained — Codecademy](https://www.codecademy.com/article/osi-model-complete-guide-to-the-7-network-layers)
- [ARIN's Vault — The IANA IPv4 Free Pool has Depleted](https://www.arin.net/vault/announcements/20110203/)
- [ThousandEyes — The DDoS Attack on Dyn's DNS Infrastructure](https://www.thousandeyes.com/blog/dyn-dns-ddos-attack)
- [DarkReading — DDoS On Dyn Used Malicious TCP, UDP Traffic](https://www.darkreading.com/cyberattacks-data-breaches/ddos-on-dyn-used-malicious-tcp-udp-traffic)
- [NERC — Lessons Learned: Networking Packet Broadcast Storms](https://www.nerc.com/globalassets/programs/event-analysis/lessons-learned/ll20181001_networking_packet_broadcast_storms.pdf)

---
**Liên kết**: [[Bai-01_Gioi-thieu-mang-may-tinh|← Bài 1]] · [[Bai-03_Phuong-tien-truyen-dan-mang|Bài 3 - Phương tiện truyền dẫn mạng →]]
