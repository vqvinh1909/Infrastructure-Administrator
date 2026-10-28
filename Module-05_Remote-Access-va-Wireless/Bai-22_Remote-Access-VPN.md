---
title: Bài 22 - Remote Access – VPN
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — VPN là công nghệ truy cập từ xa phổ biến nhất, và hiểu đúng rủi ro bảo mật liên quan (như case study Colonial Pipeline) là kiến thức thực chiến bắt buộc"
tags: [networking, cisco, vpn, ipsec, site-to-site, remote-access, sysadmin]
aliases: [VPN, IPsec, Site-to-Site VPN, Client-to-Site VPN, Remote Access VPN]
related: ["[[Bai-17_Advanced-ACL]]", "[[Bai-20_NAT]]"]
last_review: 2026-07-21
---

# Bài 22: Remote Access – VPN

> [!important] Tại sao cần file này
> Toàn bộ 21 bài trước đều giả định thiết bị nằm **trong** mạng nội bộ doanh nghiệp. Thực tế, nhân viên làm việc từ xa, chi nhánh ở tỉnh khác, hay đối tác cần truy cập 1 phần hệ thống — tất cả đều cần kết nối **an toàn** vào mạng nội bộ mà không cần kéo cáp vật lý riêng. **VPN (Virtual Private Network)** giải quyết đúng bài toán này bằng cách tạo ra 1 "đường hầm" (tunnel) mã hóa qua hạ tầng công cộng (thường là Internet).

## 1. Ý nghĩa và sự cần thiết của truy cập từ xa

Truy cập từ xa an toàn giải quyết 3 nhu cầu thực tế phổ biến nhất trong doanh nghiệp: nhân viên làm việc ngoài văn phòng vẫn cần truy cập tài nguyên nội bộ (file server, hệ thống nội bộ), chi nhánh/văn phòng vệ tinh cần kết nối liên tục với trụ sở chính như đang ở chung 1 mạng LAN, và đối tác/nhà cung cấp cần quyền truy cập giới hạn vào 1 phần hệ thống cụ thể. Nếu không có giải pháp truy cập từ xa an toàn, các nhu cầu này buộc phải "lách" bằng cách mở port trực tiếp ra Internet — rủi ro bảo mật rất lớn.

## 2. So sánh MPLS, Leased Line, và VPN

| Tiêu chí | Leased Line | MPLS | VPN (qua Internet) |
|---|---|---|---|
| Bản chất | Đường truyền vật lý riêng, dành riêng 100% cho khách hàng | Mạng do nhà mạng quản lý, dùng nhãn (label) định tuyến, chia sẻ hạ tầng nhưng cô lập logic | Tạo tunnel mã hóa qua hạ tầng Internet công cộng, dùng chung |
| Chi phí | Cao nhất | Trung bình - cao | Thấp nhất |
| Độ ổn định/độ trễ | Ổn định nhất, độ trễ thấp và có thể dự đoán | Ổn định, độ trễ có thể dự đoán tương đối tốt | Phụ thuộc hoàn toàn vào chất lượng Internet công cộng, khó dự đoán |
| Bảo mật | Cao nhất (đường riêng, không chia sẻ) | Trung bình - cao (cô lập logic nhưng vẫn qua hạ tầng chung) | Phụ thuộc vào mã hóa VPN, nhưng vẫn đi qua Internet công cộng |
| Thời gian triển khai | Lâu nhất (cần kéo cáp vật lý) | Trung bình | Nhanh nhất, chỉ cần cấu hình phần mềm/thiết bị |

> [!question] Vì sao nhiều doanh nghiệp hiện nay chọn mô hình "hybrid" (kết hợp cả VPN lẫn MPLS/Leased Line) thay vì chỉ dùng 1 loại?
> Vì mỗi công nghệ có điểm mạnh khác nhau: MPLS/Leased Line phù hợp cho kết nối **chính, ổn định lâu dài** giữa các văn phòng lớn (traffic quan trọng, cần độ trễ thấp dự đoán được), trong khi VPN qua Internet phù hợp làm **đường dự phòng chi phí thấp** hoặc phục vụ các kết nối tạm thời/số lượng lớn user từ xa không cố định (nhân viên làm việc tại nhà) — nơi việc kéo Leased Line riêng cho từng người là không khả thi về chi phí lẫn thời gian.

## 3. Mô hình VPN: Client-to-Site và Site-to-Site

```
Site-to-Site VPN (2 mang noi bo noi voi nhau qua tunnel):
+----------------+  +------------------+  +----------------+
|  TRU SO CHINH  |==|     INTERNET     |==|   CHI NHANH    |
|   Router VPN   |  | (khong tin cay)  |  |   Router VPN   |
+----------------+  +------------------+  +----------------+

Client-to-Site VPN (1 thiet bi don le noi vao mang noi bo):
+----------------+  +------------------+  +------------------+
|  TRU SO CHINH  |==|     INTERNET     |==|      Laptop      |
|   VPN Server   |  |                  |  | nhan vien tu xa  |
+----------------+  +------------------+  +------------------+
```

- **Site-to-Site VPN**: kết nối **toàn bộ 2 mạng nội bộ** với nhau qua tunnel — thiết bị ở mạng A và mạng B giao tiếp với nhau như đang ở chung 1 mạng LAN, không cần cài phần mềm VPN trên từng máy, tunnel do 2 router/firewall biên tự động xử lý.
- **Client-to-Site VPN** (còn gọi Remote Access VPN): **1 thiết bị đơn lẻ** (laptop nhân viên) chủ động thiết lập kết nối VPN vào mạng nội bộ, cần phần mềm VPN Client (ví dụ Cisco Secure Client, trước đây là AnyConnect) cài trên thiết bị đó.

## 4. Điều kiện triển khai VPN

- **Địa chỉ IP Public ổn định** ở (các) đầu router/firewall làm VPN Gateway — thường cần ít nhất 1 đầu có IP Public cố định để đầu kia khởi tạo kết nối tới.
- **Thống nhất thông số mã hóa/xác thực** giữa 2 đầu (thuật toán mã hóa, phương thức xác thực, pre-shared key hoặc chứng chỉ số) — Site-to-Site VPN yêu cầu 2 đầu khớp chính xác các thông số này.
- **NAT-Traversal** nếu 1 hoặc cả 2 đầu nằm sau thiết bị NAT ([[Bai-20_NAT|Bài 20]]) — cần cơ chế bổ sung để IPsec hoạt động đúng qua NAT (vì IPsec nguyên bản không tương thích tốt với việc địa chỉ IP bị dịch giữa đường).
- **Băng thông và độ trễ đủ đáp ứng** nhu cầu sử dụng thực tế — VPN qua Internet công cộng không đảm bảo QoS như Leased Line/MPLS.

## 5. Cấu hình VPN Site-to-Site (IPsec)

**Bước 1 — ISAKMP/IKE Phase 1** (thiết lập kênh an toàn để trao đổi khóa):

```
R1(config)# crypto isakmp policy 10
R1(config-isakmp)# encryption aes 256
R1(config-isakmp)# hash sha256
R1(config-isakmp)# authentication pre-share
R1(config-isakmp)# group 14
R1(config)# crypto isakmp key Vpn@2026 address 203.0.113.20
```

**Bước 2 — IPsec Transform Set** (thuật toán mã hóa dữ liệu thật, Phase 2):

```
R1(config)# crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
```

**Bước 3 — Định nghĩa traffic cần mã hóa** (thường bằng ACL, đã học ở [[Bai-17_Advanced-ACL|Bài 17]]):

```
R1(config)# access-list 110 permit ip 10.10.0.0 0.0.255.255 20.20.0.0 0.0.255.255
```

**Bước 4 — Crypto Map, gắn tất cả lại với nhau và áp vào interface:**

```
R1(config)# crypto map VPN-MAP 10 ipsec-isakmp
R1(config-crypto-map)# set peer 203.0.113.20
R1(config-crypto-map)# set transform-set TS
R1(config-crypto-map)# match address 110
R1(config)# interface gigabitEthernet 0/1
R1(config-if)# crypto map VPN-MAP
```

Router phía chi nhánh (`203.0.113.20`) cấu hình tương tự, với `set peer` trỏ ngược lại IP của R1, và ACL đảo chiều nguồn/đích.

> [!question] Vì sao IPsec chia thành 2 Phase (ISAKMP/IKE Phase 1 và IPsec Phase 2) thay vì làm 1 bước duy nhất?
> Vì đây là mô hình bảo mật theo lớp: **Phase 1** thiết lập 1 kênh an toàn ban đầu (dùng để trao đổi khóa 1 cách bảo mật) trước khi 2 bên tin tưởng nhau đủ để trao đổi bất kỳ thông tin nhạy cảm nào khác. **Phase 2** mới thực sự thương lượng cách mã hóa **dữ liệu traffic thật** (transform-set), diễn ra bên trong kênh an toàn đã thiết lập ở Phase 1. Tách 2 phase giúp hệ thống linh hoạt hơn (có thể đổi khóa Phase 2 định kỳ mà không cần thiết lập lại toàn bộ Phase 1) và giảm rủi ro nếu 1 phase bị tấn công.

## 6. Cấu hình VPN Client-to-Site — khái niệm cơ bản

Client-to-Site VPN trên Cisco thường triển khai qua **Cisco Secure Client (trước đây là AnyConnect)** kết hợp ASA/Router hỗ trợ SSL VPN, phức tạp hơn Site-to-Site vì cần quản lý xác thực người dùng riêng lẻ (không chỉ 2 thiết bị cố định biết trước nhau). Về khái niệm cấu hình cơ bản trên ASA (chi tiết đầy đủ về ASA sẽ học ở Bài 28). Khối cấu hình dưới đây chỉ **minh họa khái niệm, chưa đầy đủ** để dựng RA VPN chạy được — còn thiếu: nạp image client lên ASA (`webvpn` → `anyconnect image ...`), `anyconnect enable`, `vpn-tunnel-protocol ssl-client` trong group-policy, và IP pool cấp địa chỉ cho client (`ip local pool` + `address-pool` trong tunnel-group):

```
ciscoasa(config)# webvpn
ciscoasa(config-webvpn)# enable outside
ciscoasa(config)# group-policy REMOTE-USERS internal
ciscoasa(config)# username nhanvien1 password Vpn@2026
ciscoasa(config)# tunnel-group REMOTE-ACCESS type remote-access
ciscoasa(config)# tunnel-group REMOTE-ACCESS general-attributes
ciscoasa(config-tunnel-general)# default-group-policy REMOTE-USERS
```

> [!question] Vì sao Client-to-Site VPN cần quản lý tài khoản người dùng riêng lẻ, trong khi Site-to-Site chỉ cần 1 pre-shared key chung?
> Vì Site-to-Site VPN kết nối **2 thiết bị cố định đã biết trước** (2 router biên) — chỉ cần 1 khóa bí mật chung xác thực đúng giữa 2 thiết bị đó là đủ. Client-to-Site VPN phục vụ **nhiều người dùng khác nhau, số lượng có thể thay đổi liên tục** (nhân viên mới, nghỉ việc, đổi mật khẩu) — cần hệ thống xác thực theo từng tài khoản cá nhân để có thể cấp/thu hồi quyền truy cập độc lập cho từng người, đồng thời truy vết được chính xác ai đã kết nối vào hệ thống tại thời điểm nào.

## 7. Case study — Colonial Pipeline (2021): Rủi ro của Legacy VPN Account thiếu MFA

> [!example] Case study — Ransomware qua 1 tài khoản VPN cũ không dùng MFA
> Tháng 5/2021, Colonial Pipeline (nhà vận hành đường ống nhiên liệu lớn nhất nước Mỹ) bị tấn công ransomware bởi nhóm DarkSide, buộc phải ngừng hoạt động toàn bộ hệ thống đường ống trong nhiều ngày, gây thiếu hụt nhiên liệu diện rộng ở khu vực Đông Nam Mỹ và phải trả khoản tiền chuộc 4.4 triệu USD. Điểm xâm nhập ban đầu được xác nhận là **1 tài khoản VPN "legacy" (cũ, không còn dùng tới nhưng chưa bị vô hiệu hóa)** — hệ thống VPN này chỉ yêu cầu **đúng 1 mật khẩu**, không có xác thực đa yếu tố (MFA). Mật khẩu của tài khoản đó được phát hiện đã bị rò rỉ trong 1 vụ lộ dữ liệu khác từ trước (rất có thể do nhân viên dùng lại cùng 1 mật khẩu ở nhiều nơi) — kẻ tấn công chỉ cần dùng lại đúng mật khẩu đó để đăng nhập VPN thành công, không cần khai thác lỗ hổng kỹ thuật phức tạp nào.
>
> **Bài học thực chiến**: đây không phải là lỗi của công nghệ VPN nói chung, mà là lỗi vận hành — 1 tài khoản VPN không còn dùng tới **phải bị vô hiệu hóa ngay**, không được để "treo" vô thời hạn; mọi tài khoản VPN truy cập từ xa **bắt buộc phải có MFA**, không chỉ dựa vào 1 lớp mật khẩu duy nhất (đúng nguyên tắc Defense in Depth sẽ học ở Module 6); và chính sách không tái sử dụng mật khẩu giữa các hệ thống cần được thực thi nghiêm túc. Nguồn: [The Hacker News — Hackers Breached Colonial Pipeline Using Compromised VPN Password](https://thehackernews.com/2021/06/hackers-breached-colonial-pipeline.html), [Dark Reading — Colonial Pipeline CEO: Ransomware Attack Started via Pilfered 'Legacy' VPN Account](https://www.darkreading.com/cyberattacks-data-breaches/colonial-pipeline-ceo-ransomware-attack-started-via-pilfered-legacy-vpn-account).

## 8. Verify & Troubleshoot

```
Router# show crypto isakmp sa
Router# show crypto ipsec sa
Router# show crypto session
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Tunnel Site-to-Site không lên (`show crypto isakmp sa` không thấy SA) | Sai pre-shared key, hoặc thông số Phase 1 (encryption/hash/group) không khớp 2 đầu | Đối chiếu chính xác từng thông số ISAKMP policy ở cả 2 router, đảm bảo giống hệt nhau |
| Phase 1 lên nhưng traffic vẫn không đi qua được tunnel | ACL định nghĩa traffic (`match address`) không khớp hoặc bị đảo ngược sai giữa 2 đầu | Đối chiếu ACL 2 đầu, đảm bảo nguồn/đích được khai báo đối xứng đúng chiều |
| Tunnel lên rồi tự rớt liên tục | Bên nào đó nằm sau NAT nhưng chưa bật NAT-Traversal, hoặc mất ổn định đường Internet | Kiểm tra `show crypto ipsec sa` xem có thông báo lỗi liên quan NAT-T; kiểm tra chất lượng đường truyền Internet 2 đầu |
| Nhân viên không đăng nhập được Client-to-Site VPN dù đúng tài khoản | Tài khoản bị khóa/hết hạn, hoặc group-policy áp sai cho tunnel-group | Kiểm tra trạng thái tài khoản trên hệ thống xác thực; đối chiếu `tunnel-group`/`group-policy` cấu hình đúng chưa |

## 9. Lab thực hành

**Môi trường**: Cisco Packet Tracer hoặc GNS3 (Packet Tracer mô phỏng IPsec hạn chế, có thể cần dùng GNS3 nếu muốn kiểm chứng đầy đủ), 2 router đóng vai trò 2 trụ sở.

1. Cấu hình đầy đủ 4 bước Site-to-Site VPN theo mục 5 giữa 2 router.
2. Xác nhận `show crypto isakmp sa` báo trạng thái `QM_IDLE` (Phase 1 thành công) và `show crypto ipsec sa` có SA cho cả 2 chiều.
3. Ping từ PC ở mạng A sang PC ở mạng B, xác nhận traffic đi qua đúng tunnel (không đi "trần" qua Internet).
4. Cố tình sửa sai pre-shared key ở 1 đầu, quan sát tunnel không hình thành được — liên hệ bảng troubleshoot mục 8.

**Kết quả kỳ vọng**: thành thạo cấu hình Site-to-Site VPN cơ bản, hiểu rõ 2 Phase của IPsec, và nắm được bài học bảo mật thực chiến từ case study Colonial Pipeline.

## 10. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| VPN luôn an toàn tuyệt đối chỉ vì có "mã hóa" | Bỏ qua rủi ro ở lớp xác thực/quản lý tài khoản (case study mục 7) | Mã hóa chỉ bảo vệ dữ liệu trên đường truyền; nếu tài khoản/khóa bị lộ, kẻ tấn công vẫn vào được hệ thống hợp lệ như người dùng thật |
| Site-to-Site và Client-to-Site chỉ khác nhau ở số lượng thiết bị kết nối | Bỏ qua khác biệt về mô hình xác thực và quản lý | Site-to-Site dùng khóa chung cố định giữa 2 thiết bị đã biết trước; Client-to-Site cần hệ thống xác thực theo từng tài khoản cá nhân (mục 6) |
| VPN luôn nhanh và ổn định như đường truyền riêng | Nhầm lẫn giữa "được mã hóa" và "được đảm bảo chất lượng dịch vụ" | VPN qua Internet công cộng không có QoS đảm bảo như Leased Line/MPLS (mục 2) |
| Chỉ cần đúng pre-shared key là tunnel sẽ lên | Bỏ qua yêu cầu khớp toàn bộ thông số Phase 1 khác (encryption, hash, DH group) | Mọi thông số ISAKMP policy phải khớp chính xác 2 đầu, không chỉ riêng pre-shared key |

## 11. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Phân biệt Site-to-Site VPN và Client-to-Site VPN.**
→ Site-to-Site kết nối toàn bộ 2 mạng nội bộ với nhau qua tunnel do 2 router/firewall biên xử lý, không cần cài phần mềm trên từng máy. Client-to-Site (Remote Access VPN) là 1 thiết bị đơn lẻ chủ động kết nối vào mạng nội bộ, cần phần mềm VPN Client và xác thực theo từng tài khoản cá nhân.

**2. Vì sao IPsec chia thành 2 Phase?**
→ Phase 1 (ISAKMP/IKE) thiết lập kênh an toàn ban đầu để trao đổi khóa. Phase 2 (IPsec) thương lượng cách mã hóa dữ liệu traffic thật, diễn ra bên trong kênh an toàn Phase 1 đã tạo — tách lớp giúp linh hoạt và an toàn hơn.

**3. NAT-Traversal cần thiết trong trường hợp nào khi triển khai VPN?**
→ Khi 1 hoặc cả 2 đầu VPN nằm sau thiết bị NAT — vì IPsec nguyên bản không tương thích tốt với việc địa chỉ IP bị dịch giữa đường, cần cơ chế NAT-T bổ sung để tunnel vẫn hoạt động đúng.

**4. Bài học chính rút ra từ case study Colonial Pipeline là gì?**
→ 1 tài khoản VPN cũ không dùng tới nhưng chưa bị vô hiệu hóa, kết hợp thiếu MFA và mật khẩu bị rò rỉ từ nơi khác (tái sử dụng mật khẩu), đã trở thành điểm xâm nhập cho ransomware — nhấn mạnh tầm quan trọng của việc vô hiệu hóa tài khoản không dùng, bắt buộc MFA cho truy cập từ xa, và không tái sử dụng mật khẩu.

**5. Vì sao MPLS thường được chọn làm kết nối chính giữa các văn phòng lớn, còn VPN qua Internet thường làm đường dự phòng?**
→ Vì MPLS có độ trễ và độ ổn định có thể dự đoán được nhờ hạ tầng do nhà mạng quản lý, phù hợp traffic quan trọng cần chất lượng dịch vụ ổn định. VPN qua Internet công cộng chi phí thấp hơn nhiều nhưng không đảm bảo QoS, phù hợp làm phương án dự phòng hoặc phục vụ kết nối không cố định (nhân viên từ xa).

## 12. Tài liệu tham khảo

- [Firewall.cx — Configuring Site to Site IPSec VPN Tunnel Between Cisco Routers](https://www.firewall.cx/cisco/cisco-routers/cisco-router-site-to-site-ipsec-vpn.html)
- [Lightyear — MPLS vs Site-to-Site VPN: Network Solutions Comparison](https://lightyear.ai/tips/mpls-versus-sitetosite-vpn)
- [The Hacker News — Hackers Breached Colonial Pipeline Using Compromised VPN Password](https://thehackernews.com/2021/06/hackers-breached-colonial-pipeline.html)
- [Dark Reading — Colonial Pipeline CEO: Ransomware Attack Started via Pilfered 'Legacy' VPN Account](https://www.darkreading.com/cyberattacks-data-breaches/colonial-pipeline-ceo-ransomware-attack-started-via-pilfered-legacy-vpn-account)

---
**Liên kết**: [[Bai-21_FHRP|← Bài 21]] · [[Bai-23_Network-Load-Balancing|Bài 23 - Network Load Balancing →]]
