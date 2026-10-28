---
title: Bảng thuật ngữ viết tắt - Quản trị Hạ tầng mạng
type: Reference
last_review: 2026-07-21
---

# Bảng thuật ngữ viết tắt

Tổng hợp toàn bộ từ viết tắt xuất hiện xuyên suốt 30 bài học, nhóm theo chủ đề để dễ tra cứu. Mỗi mục ghi rõ tên đầy đủ, nghĩa tiếng Việt ngắn gọn, và bài học đầu tiên giới thiệu thuật ngữ đó (để đọc lại phần giải thích chi tiết nếu cần).

## Nền tảng mạng

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **OSI** | Open Systems Interconnection | Mô hình 7 tầng tham chiếu cho giao tiếp mạng | Bài 2 |
| **TCP/IP** | Transmission Control Protocol/Internet Protocol | Bộ giao thức 4 tầng thực tế Internet đang dùng | Bài 2 |
| **IEEE** | Institute of Electrical and Electronics Engineers | Tổ chức chuẩn hóa kỹ thuật (ban hành chuẩn 802.x) | Bài 3 |
| **RFC** | Request for Comments | Tài liệu chuẩn hóa giao thức Internet (IETF ban hành) | Bài 13 |
| **NIC** | Network Interface Card | Card mạng của thiết bị | Bài 4 |
| **UTP** | Unshielded Twisted Pair | Cáp xoắn đôi không vỏ chống nhiễu | Bài 3 |
| **LAN** | Local Area Network | Mạng cục bộ, phạm vi nhỏ (văn phòng, tòa nhà) | Bài 1 |
| **WAN** | Wide Area Network | Mạng diện rộng, kết nối nhiều địa điểm xa nhau | Bài 1 |
| **MAN** | Metropolitan Area Network | Mạng đô thị, phạm vi giữa LAN và WAN | Bài 1 |
| **IP** | Internet Protocol | Giao thức định địa chỉ và định tuyến gói tin | Bài 2 |
| **TCP** | Transmission Control Protocol | Giao thức truyền tải tin cậy, có bắt tay 3 bước | Bài 2 |
| **UDP** | User Datagram Protocol | Giao thức truyền tải không tin cậy, không bắt tay | Bài 2 |
| **ICMP** | Internet Control Message Protocol | Giao thức thông báo lỗi/kiểm tra kết nối (nền tảng của `ping`) | Bài 5 |
| **MAC** | Media Access Control | Địa chỉ vật lý tầng 2 gắn với card mạng | Bài 2 |
| **ARP** | Address Resolution Protocol | Giao thức phân giải IP sang MAC | Bài 5 |
| **VLSM** | Variable Length Subnet Mask | Chia subnet với độ dài mask khác nhau, tối ưu số IP dùng | Bài 5 |
| **FLSM** | Fixed Length Subnet Mask | Chia subnet với độ dài mask cố định, đơn giản nhưng lãng phí IP | Bài 5 |
| **CIDR** | Classless Inter-Domain Routing | Ký hiệu subnet dạng `/n`, không theo lớp A/B/C cố định | Bài 5 |
| **IANA** | Internet Assigned Numbers Authority | Tổ chức quản lý phân bổ địa chỉ IP toàn cầu | Bài 5 |
| **RIR** | Regional Internet Registry | Tổ chức phân bổ IP theo khu vực (dưới IANA) | Bài 5 |
| **SLAAC** | Stateless Address Autoconfiguration | Cơ chế IPv6 tự cấu hình địa chỉ không cần DHCP | Bài 6 |

## Switching (Module 2)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **VLAN** | Virtual LAN | Mạng LAN ảo, phân đoạn logic trên cùng switch vật lý | Bài 8 |
| **SVI** | Switch Virtual Interface | Interface ảo đại diện 1 VLAN trên switch Lớp 3 | Bài 12 |
| **DTP** | Dynamic Trunking Protocol | Giao thức Cisco tự động thương lượng chế độ trunk | Bài 9 |
| **VTP** | VLAN Trunking Protocol | Giao thức đồng bộ thông tin VLAN giữa các switch | Bài 9 |
| **ISL** | Inter-Switch Link | Chuẩn trunk độc quyền Cisco cũ (đã lỗi thời, thay bằng 802.1Q) | Bài 9 |
| **STP** | Spanning Tree Protocol | Giao thức chống loop Lớp 2 | Bài 10 |
| **RSTP** | Rapid Spanning Tree Protocol | Phiên bản STP hội tụ nhanh hơn | Bài 10 |
| **PVST+** | Per-VLAN Spanning Tree Plus | STP tính toán cây riêng cho từng VLAN (đặc trưng Cisco) | Bài 10 |
| **BPDU** | Bridge Protocol Data Unit | Gói tin STP trao đổi giữa các switch | Bài 10 |
| **LACP** | Link Aggregation Control Protocol | Giao thức chuẩn mở gộp nhiều link thành EtherChannel | Bài 10 |
| **PAgP** | Port Aggregation Protocol | Giao thức độc quyền Cisco gộp EtherChannel | Bài 10 |
| **CAM** | Content Addressable Memory | Bảng MAC-Port lưu trên switch (mục tiêu tấn công MAC Flooding) | Bài 26 |
| **CDP** | Cisco Discovery Protocol | Giao thức Cisco phát hiện thiết bị lân cận | Bài 7 |

## Routing (Module 3)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **AD** | Administrative Distance | Chỉ số độ tin cậy nguồn route, số càng nhỏ càng ưu tiên | Bài 11 |
| **IGP** | Interior Gateway Protocol | Giao thức định tuyến nội bộ 1 tổ chức (OSPF, EIGRP) | Bài 11 |
| **EGP** | Exterior Gateway Protocol | Giao thức định tuyến giữa các tổ chức khác nhau (BGP) | Bài 11 |
| **AS** | Autonomous System | Hệ thống mạng dưới 1 quyền quản trị duy nhất | Bài 11 |
| **BGP** | Border Gateway Protocol | Giao thức định tuyến giữa các AS trên Internet | Bài 11 |
| **OSPF** | Open Shortest Path First | Giao thức định tuyến động Link State, chuẩn mở | Bài 14 |
| **OSPFv2** | OSPF version 2 | Phiên bản OSPF cho IPv4 | Bài 14 |
| **OSPFv3** | OSPF version 3 | Phiên bản OSPF cho IPv6 | Bài 14 |
| **LSA** | Link State Advertisement | Gói tin OSPF quảng bá thông tin trạng thái liên kết | Bài 14 |
| **LSDB** | Link State Database | Cơ sở dữ liệu topology OSPF, mọi router trong area đều giống nhau | Bài 14 |
| **DR** | Designated Router | Router đại diện trên mạng Broadcast Multi-Access trong OSPF | Bài 14 |
| **BDR** | Backup Designated Router | Router dự phòng cho DR | Bài 14 |
| **ABR** | Area Border Router | Router OSPF nối giữa 2 area khác nhau | Bài 14 |
| **ASBR** | Autonomous System Boundary Router | Router OSPF nối ra ngoài AS (redistribute route từ nguồn khác) | Bài 14 |
| **MTU** | Maximum Transmission Unit | Kích thước gói tin tối đa 1 interface cho phép, phải khớp giữa 2 neighbor OSPF | Bài 14 |
| **EIGRP** | Enhanced Interior Gateway Routing Protocol | Giao thức định tuyến động lai (Hybrid) của Cisco | Bài 15 |
| **DUAL** | Diffusing Update Algorithm | Thuật toán tính đường đi của EIGRP | Bài 15 |
| **RD** | Reported Distance | Metric router lân cận báo cáo về đích, dùng tính Feasible Successor trong EIGRP | Bài 15 |
| **FS** | Feasible Successor | Đường dự phòng backup có sẵn trong EIGRP, dùng ngay không cần tính lại | Bài 15 |
| **SIA** | Stuck-In-Active | Sự cố EIGRP không nhận được phản hồi truy vấn kịp thời, route bị treo | Bài 15 |
| **MPLS** | Multiprotocol Label Switching | Công nghệ định tuyến theo nhãn, nhà mạng dùng làm kết nối WAN riêng | Bài 22 |

## Dịch vụ và bảo mật cơ bản (Module 4)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **ACL** | Access Control List | Danh sách luật lọc traffic theo tiêu chí (IP/Port/Protocol) | Bài 16 |
| **NTP** | Network Time Protocol | Giao thức đồng bộ thời gian giữa các thiết bị | Bài 18 |
| **CVE** | Common Vulnerabilities and Exposures | Mã định danh chuẩn cho 1 lỗ hổng bảo mật đã công bố | Bài 18 |
| **DHCP** | Dynamic Host Configuration Protocol | Giao thức tự động cấp phát địa chỉ IP | Bài 19 |
| **NAT** | Network Address Translation | Kỹ thuật dịch địa chỉ IP Private ↔ Public | Bài 20 |
| **PAT** | Port Address Translation | NAT Overload — nhiều IP Private dùng chung 1 IP Public, phân biệt bằng port | Bài 20 |

## Remote Access và Wireless (Module 5)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **FHRP** | First Hop Redundancy Protocol | Nhóm giao thức dự phòng Gateway (HSRP/VRRP/GLBP) | Bài 21 |
| **HSRP** | Hot Standby Router Protocol | Giao thức dự phòng Gateway độc quyền Cisco | Bài 21 |
| **VRRP** | Virtual Router Redundancy Protocol | Giao thức dự phòng Gateway chuẩn mở IETF | Bài 21 |
| **GLBP** | Gateway Load Balancing Protocol | Giao thức dự phòng Gateway kèm cân bằng tải, độc quyền Cisco | Bài 21 |
| **VIP** | Virtual IP | Địa chỉ IP ảo dùng chung bởi nhóm thiết bị dự phòng | Bài 21 |
| **VMAC** | Virtual MAC | Địa chỉ MAC ảo đi kèm VIP | Bài 21 |
| **AVG** | Active Virtual Gateway | Router được bầu làm đại diện trả lời ARP trong GLBP | Bài 21 |
| **AVF** | Active Virtual Forwarder | Router được gán forward traffic thực tế trong GLBP | Bài 21 |
| **VPN** | Virtual Private Network | Mạng riêng ảo, tạo tunnel mã hóa qua hạ tầng công cộng | Bài 22 |
| **IPsec** | Internet Protocol Security | Bộ giao thức mã hóa/xác thực traffic IP, nền tảng VPN Site-to-Site | Bài 22 |
| **ISAKMP** | Internet Security Association and Key Management Protocol | Giao thức quản lý khóa, dùng ở Phase 1 IPsec | Bài 22 |
| **IKE** | Internet Key Exchange | Giao thức trao đổi khóa an toàn trong IPsec Phase 1 | Bài 22 |
| **PSK** | Pre-Shared Key | Khóa bí mật chia sẻ trước giữa 2 bên (dùng trong IPsec, Wifi WPA) | Bài 22 |
| **NAT-T** | NAT Traversal | Cơ chế cho phép IPsec hoạt động đúng khi 1 đầu nằm sau NAT | Bài 22 |
| **PBR** | Policy Based Routing | Định tuyến theo policy tùy chỉnh, không theo routing table thông thường | Bài 23 |
| **ECMP** | Equal-Cost Multi-Path | Cân bằng tải tự động khi có nhiều route cùng cost trong routing table | Bài 23 |
| **BSS** | Basic Service Set | 1 Access Point cùng các Client kết nối vào nó | Bài 24 |
| **IBSS** | Independent BSS | Mô hình Wifi Ad-hoc, không cần Access Point | Bài 24 |
| **ESS** | Extended Service Set | Nhiều BSS nối qua Distribution System, dùng chung 1 SSID | Bài 24 |
| **DS** | Distribution System | Hạ tầng có dây kết nối nhiều Access Point với nhau | Bài 24 |
| **SSID** | Service Set Identifier | Tên mạng Wifi hiển thị cho người dùng | Bài 24 |
| **WEP** | Wired Equivalent Privacy | Chuẩn bảo mật Wifi đời đầu, đã bị phá vỡ hoàn toàn | Bài 24 |
| **WPA/WPA2/WPA3** | Wi-Fi Protected Access | Các thế hệ chuẩn bảo mật Wifi kế tiếp WEP | Bài 24 |
| **SAE** | Simultaneous Authentication of Equals | Cơ chế bắt tay của WPA3, chống dò mật khẩu offline | Bài 24 |
| **MIMO** | Multiple Input Multiple Output | Kỹ thuật dùng nhiều anten thu/phát đồng thời, tăng tốc độ Wifi | Bài 24 |
| **MU-MIMO** | Multi-User MIMO | MIMO phục vụ nhiều người dùng đồng thời | Bài 24 |
| **OFDMA** | Orthogonal Frequency-Division Multiple Access | Kỹ thuật chia nhỏ kênh sóng cho nhiều thiết bị dùng đồng thời (Wifi 6) | Bài 24 |
| **CAPWAP** | Control And Provisioning of Wireless Access Points | Giao thức chuẩn AP giao tiếp với WLC | Bài 25 |
| **LWAPP** | Lightweight Access Point Protocol | Giao thức tiền thân độc quyền Cisco trước CAPWAP | Bài 25 |
| **WLC** | Wireless LAN Controller | Thiết bị quản trị tập trung cho hạ tầng Access Point | Bài 25 |
| **DTLS** | Datagram Transport Layer Security | Mã hóa TLS cho giao thức UDP, dùng bảo vệ kênh CAPWAP | Bài 25 |
| **RRM** | Radio Resource Management | Cơ chế WLC tự động điều phối kênh sóng/công suất AP | Bài 25 |

## Bảo mật nâng cao (Module 6)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **DoS** | Denial-of-Service | Tấn công từ chối dịch vụ, làm hệ thống quá tải | Bài 26 |
| **DDoS** | Distributed Denial-of-Service | DoS phân tán từ nhiều nguồn (botnet) cùng lúc | Bài 26 |
| **MITM** | Man-in-the-Middle | Tấn công trung gian, chen vào giữa 2 bên giao tiếp | Bài 26 |
| **MD5 / SHA / AES** | Message Digest 5 / Secure Hash Algorithm / Advanced Encryption Standard | Các thuật toán băm/mã hóa dùng trong xác thực và bảo mật dữ liệu | Bài 26-27 |
| **SNMP** | Simple Network Management Protocol | Giao thức giám sát/quản lý thiết bị mạng từ xa | Bài 26 |
| **SSH** | Secure Shell | Giao thức truy cập quản trị từ xa có mã hóa | Bài 27 |
| **VTY** | Virtual Teletype | Line ảo dùng cho kết nối quản trị từ xa (Telnet/SSH) trên thiết bị Cisco | Bài 27 |
| **CLI** | Command Line Interface | Giao diện dòng lệnh quản trị thiết bị | Bài 27 |
| **AAA** | Authentication, Authorization, Accounting | Mô hình xác thực – phân quyền – ghi nhận truy cập | Bài 27 |
| **RADIUS** | Remote Authentication Dial-In User Service | Giao thức AAA chuẩn mở, dùng UDP | Bài 27 |
| **TACACS+** | Terminal Access Controller Access-Control System Plus | Giao thức AAA độc quyền Cisco, dùng TCP, mã hóa toàn bộ gói tin | Bài 27 |
| **802.1X** | IEEE 802.1X | Chuẩn xác thực truy cập mạng theo port | Bài 27 |
| **EAP** | Extensible Authentication Protocol | Giao thức xác thực dùng trong 802.1X | Bài 27 |
| **EAPOL** | EAP over LAN | EAP đóng gói truyền qua LAN giữa Supplicant và Authenticator | Bài 27 |
| **DAI** | Dynamic ARP Inspection | Cơ chế kiểm tra và chặn ARP giả mạo trên switch | Bài 26 |
| **MFA** | Multi-Factor Authentication | Xác thực đa yếu tố, thêm lớp bảo vệ ngoài mật khẩu | Bài 22, 26 |
| **ASA** | Adaptive Security Appliance | Dòng thiết bị Firewall chuyên dụng của Cisco (lưu ý: dòng ASA 5500-X đã End-of-Sale; hiện nay Cisco dùng Secure Firewall — FTD hoặc ASA chạy trên Firepower 1000/2100; giáo trình dùng ASA chỉ cho mục đích lab) | Bài 28 |
| **DMZ** | Demilitarized Zone | Vùng mạng trung gian đặt server công khai, cách ly Internet và mạng nội bộ | Bài 28 |
| **NGFW** | Next-Generation Firewall | Firewall thế hệ mới, tích hợp DPI và nhận diện ứng dụng | Bài 28 |
| **IDS** | Intrusion Detection System | Hệ thống phát hiện xâm nhập, chỉ cảnh báo (out-of-band) | Bài 28 |
| **IPS** | Intrusion Prevention System | Hệ thống ngăn chặn xâm nhập, chủ động chặn (inline) | Bài 28 |

## Automation và Programmability (Module 7)

| Viết tắt | Tên đầy đủ | Ý nghĩa | Bài |
|---|---|---|---|
| **SDN** | Software-Defined Networking | Kiến trúc tách Control Plane khỏi thiết bị, tập trung vào Controller | Bài 29 |
| **API** | Application Programming Interface | Giao diện lập trình cho phép 2 hệ thống trao đổi dữ liệu/lệnh | Bài 29 |
| **REST** | Representational State Transfer | Kiểu kiến trúc API phổ biến, dùng HTTP Verb (GET/POST/PUT/DELETE) | Bài 29 |
| **CRUD** | Create, Read, Update, Delete | 4 thao tác cơ bản với dữ liệu, tương ứng POST/GET/PUT/DELETE | Bài 29 |
| **JSON** | JavaScript Object Notation | Định dạng dữ liệu key-value, phổ biến nhất cho REST API | Bài 29 |
| **XML** | Extensible Markup Language | Định dạng dữ liệu dùng thẻ mở/đóng, phổ biến trong NETCONF | Bài 29 |
| **YAML** | YAML Ain't Markup Language | Định dạng dữ liệu dựa trên thụt lề, dùng trong Ansible Playbook | Bài 29 |
| **ZTP** | Zero Touch Provisioning | Cơ chế thiết bị tự tải cấu hình khi kết nối lần đầu, không cần cấu hình tay | Bài 29 |
