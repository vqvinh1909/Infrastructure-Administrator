---
title: Bài 26 - Các nguy cơ về an toàn bảo mật – Phòng tránh
type: Concept
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — nhận diện đúng loại tấn công (Spoofing/DoS/MITM/Buffer Overflow/Malware/Password/Human) là kiến thức nền bắt buộc trước khi thiết kế bất kỳ biện pháp phòng thủ nào, và là nhóm câu hỏi phỏng vấn bảo mật kinh điển nhất"
tags: [networking, security, spoofing, dos, mitm, malware, social-engineering, port-security, dhcp-snooping, arp-inspection, sysadmin]
aliases: [Spoofing, DoS, DDoS, MITM, Buffer Overflow, Malware, Social Engineering, Port Security, DHCP Snooping, Dynamic ARP Inspection, BPDU Guard, Root Guard]
related: ["[[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco]]", "[[Bai-10_STP-va-EtherChannel]]", "[[Bai-19_Dich-vu-DHCP]]", "[[Bai-22_Remote-Access-VPN]]", "[[Bai-27_AAA]]", "[[Bai-28_Cisco-ASA-Firewall]]"]
last_review: 2026-07-21
---

# Bài 26: Các nguy cơ về an toàn bảo mật – Phòng tránh

> [!important] Tại sao cần file này
> Trước khi học cách **phòng thủ** (Bài 27, Bài 28), cần hiểu rõ mình đang phòng thủ trước **cái gì**. Bài này chia làm 2 phần: phần đầu (mục 1-9) hệ thống hóa các nhóm nguy cơ bảo mật phổ biến nhất mà hạ tầng mạng doanh nghiệp phải đối mặt — Spoofing, DoS, MITM, Buffer Overflow, Malware, lỗ hổng mật khẩu, lỗ hổng con người; phần sau (mục 10-13) đi sâu vào các biện pháp phòng thủ **cụ thể ở Lớp 2** (Port Security, DHCP Snooping, Dynamic ARP Inspection, bảo mật STP) — chính là công cụ thực chiến để ngăn chặn nhiều nguy cơ đã nêu ở phần đầu.

## 1. Phân loại các mối đe dọa bảo mật phổ biến

| Nhóm nguy cơ | Bản chất | Nhắm vào yếu tố nào |
|---|---|---|
| **Spoofing** (giả mạo địa chỉ) | Giả mạo địa chỉ MAC/IP/ARP để đội lốt thiết bị hợp lệ | Danh tính thiết bị trên mạng |
| **DoS/DDoS** | Làm hệ thống quá tải, không thể phục vụ người dùng hợp lệ | Tính sẵn sàng (Availability) |
| **Man-in-the-Middle** | Chen vào giữa 2 bên giao tiếp để nghe lén/chỉnh sửa traffic | Tính bí mật/toàn vẹn (Confidentiality/Integrity) |
| **Buffer Overflow** | Khai thác lỗi quản lý bộ nhớ của phần mềm/hệ điều hành | Lỗ hổng ở tầng ứng dụng/hệ thống |
| **Malware** | Phần mềm độc hại tự lây lan hoặc được cài cắm | Thiết bị đầu cuối, dữ liệu |
| **Password Vulnerabilities** | Khai thác mật khẩu yếu/bị lộ | Xác thực (Authentication) |
| **Human Vulnerabilities** | Khai thác tâm lý con người thay vì lỗ hổng kỹ thuật | Yếu tố con người trong hệ thống |

Đây không phải danh sách tách biệt hoàn toàn — trong thực tế, 1 cuộc tấn công thường **kết hợp nhiều nhóm** (ví dụ: Phishing khai thác Human Vulnerabilities để lấy mật khẩu, sau đó mật khẩu bị lộ đó lại là 1 Password Vulnerability bị khai thác tiếp — chính là mô hình xảy ra trong case study Colonial Pipeline ở [[Bai-22_Remote-Access-VPN|Bài 22]] mục 7).

## 2. Attacks That Spoof Addresses — Tấn công giả mạo địa chỉ

**Spoofing** là kỹ thuật giả mạo địa chỉ (MAC, IP, hoặc ánh xạ ARP) để hệ thống nhận nhầm kẻ tấn công là 1 thiết bị/nguồn hợp lệ.

| Loại Spoofing | Cơ chế | Mục đích thường gặp |
|---|---|---|
| **MAC Address Spoofing** | Đổi địa chỉ MAC khai báo trên card mạng của thiết bị tấn công thành trùng với 1 MAC hợp lệ đã biết | Vượt qua kiểm soát truy cập dựa trên MAC (như Port Security ở chế độ chỉ kiểm MAC, không xác thực danh tính thật) |
| **IP Address Spoofing** | Giả mạo địa chỉ IP nguồn trong header gói tin gửi đi | Che giấu nguồn gốc thật khi tấn công DoS (mục 3), hoặc vượt qua ACL chỉ lọc theo IP nguồn |
| **ARP Spoofing (ARP Poisoning)** | Gửi ARP Reply giả, đánh lừa các thiết bị khác ghi nhận sai ánh xạ IP-MAC | Thường dùng làm bước mở đầu cho tấn công Man-in-the-Middle (mục 4) |
| **DHCP Spoofing** | Dựng 1 DHCP server giả (rogue) phát IP/Gateway sai cho nạn nhân | Dẫn traffic nạn nhân đi qua thiết bị kẻ tấn công (đã học chi tiết ở [[Bai-19_Dich-vu-DHCP|Bài 19]]) |

> [!question] Vì sao MAC Address Spoofing có thể vượt qua Port Security dù Port Security được thiết kế để kiểm soát chính xác địa chỉ MAC?
> Vì địa chỉ MAC **không phải là bí mật** — nó được truyền công khai trong mọi khung Ethernet và dễ dàng đọc được chỉ bằng cách bắt gói tin (packet capture) trên cùng đoạn mạng. Port Security chỉ kiểm tra "địa chỉ MAC xuất hiện trên port này có nằm trong danh sách cho phép không", chứ không xác minh **danh tính thật** đứng sau địa chỉ MAC đó. Nếu kẻ tấn công biết trước 1 MAC hợp lệ (qua bắt gói tin) và đổi MAC card mạng của mình thành đúng giá trị đó, Port Security sẽ không phát hiện được gì bất thường. Đây chính là lý do 802.1X ([[Bai-27_AAA|Bài 27]] mục 4) — xác thực bằng thông tin định danh thật thay vì chỉ dựa vào MAC — được xem là lớp kiểm soát mạnh hơn.

## 3. Denial-of-Service Attacks — Tấn công từ chối dịch vụ

**DoS** (Denial-of-Service) làm hệ thống/dịch vụ **quá tải hoặc không thể phản hồi**, khiến người dùng hợp lệ không thể sử dụng được — khác với các tấn công khác thường nhắm tới đánh cắp dữ liệu, DoS nhắm thẳng vào **tính sẵn sàng** (Availability) của hệ thống.

| Kỹ thuật | Cơ chế |
|---|---|
| **DoS** (1 nguồn tấn công) | 1 máy duy nhất gửi lượng lớn traffic/request vượt khả năng xử lý của mục tiêu |
| **DDoS** (Distributed DoS, nhiều nguồn) | Hàng nghìn thiết bị bị chiếm quyền điều khiển (botnet) cùng tấn công 1 mục tiêu, khó chặn hơn nhiều vì traffic đến từ vô số địa chỉ IP khác nhau |
| **SYN Flood** | Gửi hàng loạt gói TCP SYN nhưng không hoàn tất bắt tay 3 bước, làm cạn kiệt tài nguyên bảng kết nối "half-open" của server |
| **Amplification Attack** | Lợi dụng dịch vụ có phản hồi lớn hơn nhiều lần so với request (như NTP, DNS) kết hợp giả mạo IP nguồn thành IP nạn nhân — đã học cụ thể qua case study NTP Amplification DDoS 2014 ở [[Bai-18_NTP-va-Syslog|Bài 18]] |

> [!question] Vì sao DDoS khó phòng chống hơn DoS thông thường, dù về bản chất cả 2 đều nhằm làm quá tải mục tiêu?
> Vì với DoS từ 1 nguồn, chỉ cần xác định đúng địa chỉ IP đang tấn công rồi chặn (qua ACL/Firewall) là có thể ngăn chặn hiệu quả. Với DDoS, traffic tấn công đến từ **hàng nghìn địa chỉ IP hợp lệ khác nhau** (các thiết bị trong botnet, thường là nạn nhân bị nhiễm malware mà không hề biết) — không thể chỉ chặn 1 vài IP cụ thể mà không ảnh hưởng tới traffic hợp lệ khác, vì bản thân từng gói tin trông hoàn toàn "hợp lệ" khi xét riêng lẻ. Phòng chống DDoS hiệu quả thường cần dịch vụ chuyên dụng (DDoS scrubbing/CDN) có khả năng phân tích baseline traffic ở quy mô lớn để phân biệt traffic tấn công với traffic thật.

## 4. Man-in-the-Middle Attacks — Tấn công trung gian

**MITM** (Man-in-the-Middle) là kỹ thuật kẻ tấn công **chen vào giữa** 2 bên đang giao tiếp mà cả 2 bên không hề hay biết — cho phép nghe lén, hoặc thậm chí **chỉnh sửa** nội dung traffic đang truyền qua.

| Vector tấn công | Cơ chế |
|---|---|
| **ARP Spoofing** (Lớp 2, đã nêu ở mục 2) | Đánh lừa 2 thiết bị trong cùng subnet gửi traffic qua đúng máy kẻ tấn công trước khi tới đích thật |
| **Rogue Access Point** (Wifi) | Dựng 1 AP giả mạo tên SSID hợp lệ, dụ thiết bị nạn nhân kết nối nhầm (liên quan tới kiến trúc Wifi đã học ở [[Bai-24_Wireless-LAN|Bài 24]]) |
| **Session Hijacking** | Đánh cắp thông tin phiên làm việc (session token/cookie) đã xác thực để giả danh người dùng hợp lệ mà không cần biết mật khẩu |

> [!question] Vì sao mã hóa (HTTPS/TLS, VPN) lại là biện pháp phòng chống MITM hiệu quả nhất, dù không ngăn được việc kẻ tấn công "chen vào giữa"?
> Vì các biện pháp như DAI (mục 12) chỉ ngăn được **1 vector cụ thể** (ARP Spoofing ở Lớp 2) — không ngăn được mọi hình thức MITM (ví dụ Rogue AP không liên quan gì tới ARP). Mã hóa đầu-cuối tấn công vào **hậu quả chung** của MITM: dù kẻ tấn công có chen được vào giữa đường truyền, dữ liệu đi qua đã được mã hóa (như IPsec VPN ở [[Bai-22_Remote-Access-VPN|Bài 22]], hoặc WPA2/WPA3 ở [[Bai-24_Wireless-LAN|Bài 24]] mục 4) khiến kẻ tấn công **chỉ thấy dữ liệu vô nghĩa** dù đã đứng đúng vị trí "giữa đường". Đây là lý do phòng thủ MITM hiệu quả nhất luôn kết hợp cả ngăn chặn vector cụ thể (DAI, chặn Rogue AP) lẫn mã hóa dữ liệu như lớp bảo vệ cuối cùng.

## 5. Buffer Overflow Attacks — Tấn công tràn bộ đệm

**Buffer** (bộ đệm) là vùng nhớ có kích thước cố định mà 1 chương trình dùng để lưu tạm dữ liệu đang xử lý. **Buffer Overflow** xảy ra khi chương trình nhận dữ liệu đầu vào **lớn hơn** kích thước buffer đã cấp phát nhưng không kiểm tra/giới hạn đúng — phần dữ liệu dư ra sẽ "tràn" sang vùng nhớ liền kề, ghi đè lên dữ liệu hoặc mã lệnh đang có ở đó. Trong trường hợp nghiêm trọng nhất, kẻ tấn công có thể cố tình chèn **mã lệnh độc hại** vào đúng phần dữ liệu tràn, khiến chương trình vô tình thực thi mã đó — cho phép chiếm quyền điều khiển thiết bị.

> [!question] Buffer Overflow chủ yếu là lỗi lập trình ứng dụng — vậy vì sao Sysadmin vẫn cần quan tâm tới loại tấn công này?
> Vì dù nguyên nhân gốc rễ nằm ở việc lập trình viên không kiểm tra/giới hạn đúng kích thước dữ liệu đầu vào, **hậu quả** của lỗ hổng đó lại đổ lên vai Sysadmin — thiết bị mạng (router, switch, firewall) cũng chạy phần mềm (IOS, ASA OS...) và **cũng có thể chứa lỗ hổng Buffer Overflow** như bất kỳ phần mềm nào khác (Cisco vẫn định kỳ công bố các bản vá cho lỗ hổng dạng này trên chính thiết bị của mình). Trách nhiệm của Sysadmin không phải là sửa lỗi code, mà là: duy trì **Patch Management** nghiêm túc (cập nhật bản vá kịp thời khi nhà sản xuất công bố), và triển khai **IPS** ([[Bai-28_Cisco-ASA-Firewall|Bài 28]] mục 5) có khả năng nhận diện signature của các payload khai thác Buffer Overflow đã biết để chặn trước khi nó tới được thiết bị dễ tổn thương.

## 6. Malware — Phần mềm độc hại

| Loại Malware | Cơ chế lây lan | Đặc điểm |
|---|---|---|
| **Virus** | Gắn vào 1 phần mềm/file khác, chỉ lây khi phần mềm/file đó được chạy | Cần "vật chủ" để tồn tại và lây lan |
| **Worm** | **Tự lây lan** qua mạng, không cần người dùng tương tác hay có file vật chủ | Có thể lan rộng cực nhanh trong toàn hệ thống mạng |
| **Trojan** (Trojan Horse) | Giả dạng phần mềm hợp lệ, người dùng tự tay cài đặt mà không biết | Không tự nhân bản, cần người dùng chủ động thực thi |
| **Ransomware** | Mã hóa dữ liệu nạn nhân, đòi tiền chuộc để khôi phục | Case study điển hình: Colonial Pipeline 2021 (đã học ở [[Bai-22_Remote-Access-VPN|Bài 22]] mục 7) |
| **Spyware** | Ẩn mình theo dõi hoạt động, đánh cắp thông tin nhạy cảm (mật khẩu, dữ liệu tài chính) | Thường không làm gián đoạn hoạt động thiết bị, khó bị phát hiện |

> [!question] Vì sao Worm được xem là nguy hiểm hơn Virus xét về tốc độ lan rộng trong mạng doanh nghiệp?
> Vì Virus phụ thuộc vào hành vi con người (phải mở/chạy đúng file/phần mềm nhiễm) để lây lan — tốc độ lan truyền bị giới hạn bởi tần suất người dùng chia sẻ/mở file đó. Worm **tự chủ động tìm kiếm và lây lan** qua chính hạ tầng mạng (khai thác lỗ hổng dịch vụ đang chạy, quét dải IP nội bộ) mà không cần bất kỳ hành động nào từ con người — trong môi trường doanh nghiệp có hàng trăm thiết bị kết nối cùng mạng nội bộ, 1 Worm khai thác đúng lỗ hổng phổ biến có thể lây lan tới toàn bộ hệ thống chỉ trong vài phút, nhanh hơn nhiều so với tốc độ con người có thể phản ứng.

## 7. Password Vulnerabilities — Các lỗ hổng mật khẩu

| Lỗ hổng | Cơ chế khai thác |
|---|---|
| **Mật khẩu yếu/mặc định** | Dùng mật khẩu ngắn, dễ đoán, hoặc chưa đổi khỏi giá trị mặc định của nhà sản xuất |
| **Brute Force Attack** | Thử tuần tự mọi tổ hợp ký tự có thể cho tới khi đúng |
| **Dictionary Attack** | Thử danh sách mật khẩu phổ biến/từ điển thay vì mọi tổ hợp ngẫu nhiên, hiệu quả hơn Brute Force thuần túy |
| **Credential Stuffing / Tái sử dụng mật khẩu** | Dùng lại cặp username/password đã bị lộ từ 1 vụ rò rỉ dữ liệu khác để đăng nhập thử vào hệ thống khác | 

Case study Colonial Pipeline ([[Bai-22_Remote-Access-VPN|Bài 22]] mục 7) là minh chứng trực tiếp cho **Credential Stuffing**: mật khẩu VPN bị lộ từ 1 vụ rò rỉ dữ liệu hoàn toàn khác trước đó, nhiều khả năng do nhân viên tái sử dụng cùng 1 mật khẩu ở nhiều nơi.

> [!question] Vì sao chỉ yêu cầu mật khẩu "đủ dài, đủ phức tạp" là chưa đủ để phòng chống Password Vulnerabilities một cách toàn diện?
> Vì độ dài/độ phức tạp chỉ giúp chống lại **Brute Force/Dictionary Attack** — nơi kẻ tấn công phải "đoán" mật khẩu từ đầu. Nó hoàn toàn không giúp ích gì trước **Credential Stuffing**, vì trong trường hợp đó mật khẩu đã bị lộ **nguyên vẹn, đúng 100%** từ 1 hệ thống khác — dù mật khẩu đó dài và phức tạp tới đâu cũng vô nghĩa nếu kẻ tấn công đã có sẵn nó. Đây là lý do phòng thủ toàn diện cần thêm lớp **MFA** (xác thực đa yếu tố, liên quan tới AAA đã học ở [[Bai-27_AAA|Bài 27]]) — ngay cả khi mật khẩu đúng bị lộ, kẻ tấn công vẫn cần thêm yếu tố thứ 2 (mã OTP, thiết bị xác thực vật lý) mà chúng không có, và chính sách **không tái sử dụng mật khẩu** giữa các hệ thống khác nhau.

## 8. Human Vulnerabilities — Các lỗ hổng của con người

Khác với các nhóm trên (khai thác lỗ hổng kỹ thuật), **Social Engineering** khai thác trực tiếp **tâm lý con người** — thường là cách hiệu quả nhất để vượt qua mọi lớp phòng thủ kỹ thuật đã đầu tư, vì nó nhắm vào mắt xích không thể "vá lỗi" bằng phần mềm.

| Kỹ thuật | Cơ chế |
|---|---|
| **Phishing** | Gửi email/tin nhắn giả mạo nguồn đáng tin cậy, dụ nạn nhân bấm link độc hại hoặc cung cấp thông tin nhạy cảm |
| **Pretexting** | Dựng 1 "kịch bản"/tình huống giả (giả danh nhân viên cấp cao, đối tác) để xây dựng lòng tin trước khi khai thác thông tin |
| **Baiting** | Đặt "mồi nhử" hấp dẫn (USB miễn phí, file tải hấp dẫn) để dụ nạn nhân tự tay kích hoạt malware |
| **Tailgating** | Bám theo người có quyền truy cập hợp lệ để lẻn vào khu vực vật lý được kiểm soát, không cần thẻ ra vào riêng |
| **Quid Pro Quo** | Giả danh hỗ trợ kỹ thuật (IT Support), đề nghị "giúp đỡ" đổi lấy thông tin đăng nhập của nạn nhân |

> [!question] Vì sao Social Engineering thường được xem là vector tấn công hiệu quả nhất, vượt qua mọi lớp phòng thủ kỹ thuật đã học ở Bài 26-28?
> Vì mọi biện pháp kỹ thuật đã học (Port Security, DHCP Snooping, DAI, AAA, Firewall) đều giả định **kẻ tấn công là bên ngoài, chưa có quyền truy cập hợp lệ**. Social Engineering đi vòng qua toàn bộ giả định đó bằng cách khiến chính **người dùng hợp lệ** (người đã có sẵn quyền truy cập, mật khẩu đúng, thẻ ra vào thật) tự nguyện thực hiện hành động có lợi cho kẻ tấn công — không có tường lửa hay ACL nào ngăn được 1 nhân viên tự tay nhập đúng mật khẩu của mình vào 1 trang web giả mạo trông giống hệt trang thật. Đây là lý do phòng thủ trước Social Engineering không thể chỉ dựa vào công nghệ, mà bắt buộc cần **đào tạo nhận thức bảo mật (security awareness training)** định kỳ cho toàn bộ nhân viên.

## 9. Phương thức phòng tránh, giảm thiểu nguy cơ và thiệt hại

Không có 1 công cụ/thiết bị duy nhất phòng chống được toàn bộ nhóm nguy cơ vừa nêu — đây chính là nguyên tắc **Defense in Depth** (đã tổng hợp chi tiết qua case study Colonial Pipeline ở [[Bai-22_Remote-Access-VPN|Bài 22]] mục 7, và bảng tổng hợp đầy đủ các lớp phòng thủ theo từng bài học ở [[Bai-28_Cisco-ASA-Firewall|Bài 28]] mục 6). Với riêng nhóm nguy cơ xảy ra ở **Lớp 2** — nơi Spoofing địa chỉ (mục 2) và bước mở đầu của nhiều tấn công MITM (mục 4) thực sự diễn ra — switch access chính là **lớp phòng thủ đầu tiên**, gần với người dùng cuối nhất. Mọi thiết kế bảo mật Lớp 3 (firewall, ACL, VPN) đều giả định traffic đã đi vào đúng đường mong muốn; nếu switch access không được bảo vệ, kẻ tấn công cắm cáp vào 1 port trống có thể bỏ qua toàn bộ các lớp phòng thủ phía trên. Từ mục 10 trở đi, bài học đi sâu vào 4 biện pháp phòng thủ **cụ thể, có thể cấu hình được ngay** ở Lớp 2.

## 10. Port Security — Giới hạn MAC address trên từng port

**Cơ chế**: Port Security giới hạn số lượng địa chỉ MAC được phép học trên 1 switchport, và/hoặc chỉ cho phép đúng (các) MAC cụ thể — ngăn kẻ tấn công cắm thêm thiết bị lạ vào port, hoặc tấn công **MAC Flooding** (làm tràn bảng CAM của switch bằng hàng loạt MAC giả để buộc switch chuyển sang chế độ "fail-open" — gửi broadcast toàn bộ traffic ra mọi port thay vì unicast đúng đích, cho phép kẻ tấn công nghe lén traffic vốn không dành cho mình).

| Chế độ học MAC | Cơ chế |
|---|---|
| **Static** | Quản trị viên khai báo cứng (các) MAC được phép, không tự học |
| **Dynamic** | Switch tự học MAC đầu tiên xuất hiện trên port, nhưng **mất khi port down/switch reload** |
| **Sticky** | Switch tự học MAC như Dynamic, nhưng **tự động lưu vào running-config** — nếu `copy running-config startup-config` thì tồn tại vĩnh viễn qua reboot |

**Chế độ xử lý vi phạm (violation mode)** khi có MAC lạ xuất hiện vượt giới hạn:

| Chế độ | Hành vi | Có ghi log/SNMP trap không |
|---|---|---|
| **Protect** | Chặn traffic từ MAC lạ, vẫn cho MAC hợp lệ hoạt động bình thường | Không |
| **Restrict** | Giống Protect, nhưng có ghi log/SNMP trap và tăng bộ đếm vi phạm | Có |
| **Shutdown** (mặc định) | Đưa port vào trạng thái **err-disabled**, ngừng hoạt động hoàn toàn cho tới khi quản trị viên can thiệp thủ công | Có |

```
Switch(config)# interface fastEthernet 0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
Switch(config-if)# switchport port-security mac-address sticky
Switch(config-if)# switchport port-security violation restrict
```

> [!question] Vì sao `shutdown` là chế độ mặc định dù nó gây gián đoạn hoàn toàn cho người dùng hợp lệ trên port đó?
> Vì triết lý bảo mật ở đây là "an toàn trước, tiện lợi sau" (fail-secure) — khi có dấu hiệu bất thường (MAC lạ xuất hiện), hệ thống mặc định chọn phương án **chắc chắn ngăn chặn hoàn toàn** thay vì chỉ lọc 1 phần traffic (như Protect/Restrict) — bởi vì tại thời điểm phát hiện, switch không thể biết chắc đây là thiết bị mới hợp lệ (nhân viên đổi laptop) hay đang bị tấn công. Đưa port vào err-disabled buộc quản trị viên phải **chủ động xác minh** trước khi khôi phục, thay vì để hệ thống tự "đoán" và có thể đoán sai theo hướng nguy hiểm hơn.

## 11. DHCP Snooping — Chặn DHCP Server giả mạo

**Cơ chế**: DHCP Snooping phân loại mỗi switchport thành **Trusted** (tin cậy — nơi đặt DHCP server hợp lệ, hoặc port uplink hướng lên switch/router khác) hoặc **Untrusted** (mặc định cho mọi port — thường là port nối tới PC người dùng). Trên port Untrusted, switch chỉ cho phép **DHCP Request** đi qua (traffic PC gửi lên xin IP) — nếu switch phát hiện gói tin **DHCP Reply/Offer/ACK** (traffic chỉ server hợp lệ mới được gửi) xuất hiện trên port Untrusted, gói tin đó bị chặn (hoặc port bị đưa vào err-disabled tùy cấu hình) — vì đó là dấu hiệu có 1 DHCP server giả mạo (rogue DHCP server, đã nêu ở mục 2) đang cắm vào đúng port đó.

```
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# ip dhcp snooping trust
```

Interface `Gi0/1` (thường là uplink hướng tới router/DHCP server thật) được đánh dấu **Trusted**; mọi port còn lại (hướng tới PC) mặc định **Untrusted** và tự động được bảo vệ ngay khi bật `ip dhcp snooping` cho VLAN tương ứng.

DHCP Snooping còn tạo ra **Binding Table** — bảng ghi lại ánh xạ (MAC, IP, VLAN, port) của từng thiết bị đã nhận IP hợp lệ qua DHCP, dùng làm dữ liệu tham chiếu cho Dynamic ARP Inspection (mục 12).

## 12. Dynamic ARP Inspection (DAI) — Chặn ARP giả mạo

**Cơ chế**: ARP không có xác thực — bất kỳ thiết bị nào cũng có thể gửi ARP Reply giả (kỹ thuật **ARP Spoofing/Poisoning**, đã nêu ở mục 2), khiến các thiết bị khác trong cùng subnet ghi nhận sai ánh xạ IP-MAC, dẫn traffic (kể cả traffic tới Gateway) đi qua máy kẻ tấn công trước (chính là bước mở đầu phổ biến nhất của tấn công Man-in-the-Middle, mục 4). DAI kiểm tra mọi gói ARP đi qua port **Untrusted**, đối chiếu với **Binding Table** đã tạo bởi DHCP Snooping (mục 11) — nếu thông tin (IP, MAC) trong gói ARP không khớp với bản ghi hợp lệ đã biết, gói tin bị loại bỏ ngay lập tức.

```
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# ip arp inspection trust
```

> [!question] Vì sao DAI bắt buộc phải bật DHCP Snooping trước, thay vì hoạt động độc lập?
> Vì DAI **không tự học** ánh xạ IP-MAC hợp lệ — nó chỉ so sánh gói ARP nhận được với 1 nguồn dữ liệu tham chiếu đáng tin cậy, và nguồn đó chính là **Binding Table** do DHCP Snooping xây dựng (ghi nhận thiết bị nào đã nhận đúng IP nào qua DHCP hợp lệ). Nếu không có DHCP Snooping chạy trước, DAI không có gì để đối chiếu và sẽ chặn nhầm toàn bộ ARP hợp lệ (hoặc phải chuyển sang dùng ARP ACL tĩnh, cấu hình thủ công phức tạp hơn nhiều cho mạng lớn).

## 13. Bảo mật STP — BPDU Guard và Root Guard

[[Bai-10_STP-va-EtherChannel|Bài 10]] đã học STP chống loop, nhưng STP mặc định **tin tưởng mọi switch gửi BPDU** — nếu kẻ tấn công cắm 1 switch giả (hoặc máy tính giả lập gửi BPDU) vào port access với Priority thấp hơn, nó có thể tự xưng làm **Root Bridge mới**, làm thay đổi toàn bộ topology STP của mạng và có thể gây gián đoạn lớn.

| Cơ chế | Mục đích | Hành vi khi phát hiện |
|---|---|---|
| **BPDU Guard** | Ngăn port access (nơi chỉ nên cắm PC, không phải switch khác) nhận **bất kỳ** BPDU nào | Nhận được BPDU → đưa ngay port vào **err-disabled**, không tự phục hồi |
| **Root Guard** | Ngăn 1 switch/port cụ thể trở thành đường dẫn tới Root Bridge mới có Priority tốt hơn | Nhận BPDU "tốt hơn" (superior) → đưa port vào trạng thái **root-inconsistent** (chặn traffic), **tự phục hồi** khi hết nhận BPDU tốt hơn |

```
Switch(config)# interface fastEthernet 0/1
Switch(config-if)# spanning-tree portfast
Switch(config-if)# spanning-tree bpduguard enable
```

```
Switch(config)# interface gigabitEthernet 0/2
Switch(config-if)# spanning-tree guard root
```

> [!question] Vì sao BPDU Guard đặt trên port Access (hướng PC) còn Root Guard đặt trên port hướng tới switch khác (thường là port Distribution xuống Access)?
> Vì 2 cơ chế nhắm tới 2 tình huống khác nhau. BPDU Guard giả định port đó **không nên có switch nào khác cắm vào** — chỉ cần thấy bất kỳ BPDU nào xuất hiện là đủ để coi là bất thường, phản ứng dứt khoát (khóa hẳn port). Root Guard dùng cho các port **có thể hợp lệ nhận BPDU** (vì nối tới switch khác thật), nhưng switch phía bên kia **không được phép trở thành Root Bridge** — ví dụ port từ Distribution xuống 1 switch Access ở tầng thấp hơn trong thiết kế phân lớp; nếu switch Access đó (có thể bị cấu hình sai hoặc bị tấn công) tự xưng Priority thấp để thành Root, Root Guard sẽ chặn đúng port đó thay vì để toàn mạng tính toán lại theo Root sai.

## 14. Verify & Troubleshoot

```
Switch# show port-security interface fastEthernet 0/1
Switch# show ip dhcp snooping binding
Switch# show ip arp inspection interfaces
Switch# show spanning-tree summary
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Port bị err-disabled sau khi cắm laptop mới | Port Security vi phạm chế độ `shutdown` (MAC mới vượt giới hạn cho phép) | `show port-security interface` xác nhận vi phạm; `shutdown` rồi `no shutdown` port sau khi xác minh thiết bị hợp lệ (hoặc cấu hình `errdisable recovery`) |
| PC không nhận được IP dù DHCP server vẫn hoạt động bình thường | Port hướng DHCP server chưa được đánh dấu `ip dhcp snooping trust`, khiến DHCP Snooping chặn nhầm Reply hợp lệ | Kiểm tra đúng port uplink đã có `trust` chưa (mục 11) |
| ARP hợp lệ bị DAI chặn nhầm (ví dụ IP tĩnh không qua DHCP) | DAI không tìm thấy IP đó trong Binding Table (vì thiết bị không nhận IP qua DHCP) | Thêm ARP ACL tĩnh (`arp access-list`) khai báo riêng cho các thiết bị dùng IP tĩnh, hoặc dùng `ip source binding` thủ công |
| Toàn mạng đột ngột thay đổi Root Bridge, hội tụ lại STP bất thường | Có switch/thiết bị lạ gửi BPDU với Priority thấp hơn Root hiện tại, chưa có Root Guard bảo vệ | Bật `spanning-tree guard root` trên các port không nên dẫn tới Root Bridge hợp lệ |
| Server nội bộ đột ngột không phản hồi, CPU/băng thông tăng vọt bất thường | Có thể đang là mục tiêu DoS/DDoS (mục 3), hoặc đã nhiễm Worm đang tự lây lan (mục 6) | Kiểm tra log/traffic bất thường; cô lập thiết bị nghi ngờ khỏi mạng để điều tra trước khi lan rộng |

## 15. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 1 switch, 2-3 PC, 1 DHCP server hợp lệ, 1 "DHCP server giả" mô phỏng.

1. Cấu hình Port Security theo mục 10 trên 1 port, giới hạn tối đa 1 MAC, chế độ `restrict`. Thử đổi MAC ảo của PC (hoặc cắm thêm 1 thiết bị), quan sát log vi phạm.
2. Bật DHCP Snooping theo mục 11, đánh dấu đúng port hướng DHCP server hợp lệ là `trust`. Thử mô phỏng 1 DHCP server giả ở port khác, xác nhận PC không nhận được IP từ server giả.
3. Bật DAI theo mục 12, quan sát `show ip arp inspection interfaces` xác nhận các port PC đều Untrusted.
4. Cấu hình BPDU Guard trên 1 port access, thử mô phỏng gửi BPDU vào port đó (hoặc cắm switch khác vào), xác nhận port chuyển err-disabled ngay lập tức.

**Kết quả kỳ vọng**: hiểu và tự tay kiểm chứng được 4 lớp phòng thủ Lớp 2 cơ bản, nhận diện đúng cơ chế trusted/untrusted xuyên suốt DHCP Snooping và DAI, đồng thời liên hệ được từng lớp phòng thủ với đúng nhóm nguy cơ đã học ở mục 1-9.

## 16. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Chỉ cần ACL/Firewall ở Lớp 3 là đủ bảo vệ mạng nội bộ | Bỏ qua các kỹ thuật tấn công không cần đụng tới IP | Nhiều tấn công (MAC Flooding, Rogue DHCP, ARP Spoofing) xảy ra hoàn toàn ở Lớp 2, trước khi traffic tới được thiết bị Lớp 3 (mục 9) |
| DAI hoạt động độc lập, không liên quan gì tới DHCP | Bỏ qua cơ chế DAI dựa vào Binding Table | DAI mặc định cần Binding Table do DHCP Snooping tạo ra để có dữ liệu đối chiếu hợp lệ (mục 12) |
| Root Guard và BPDU Guard là 2 tên gọi khác nhau của cùng 1 cơ chế | Nhầm lẫn do cả 2 đều liên quan BPDU | Root Guard chặn switch trở thành Root Bridge (tự phục hồi khi hết nhận BPDU tốt hơn); BPDU Guard chặn hẳn port nhận bất kỳ BPDU nào (không tự phục hồi) — mục đích và hành vi khác nhau (mục 13) |
| Buffer Overflow không liên quan tới công việc Sysadmin vì đó là lỗi lập trình | Bỏ qua trách nhiệm Patch Management và IPS của Sysadmin | Thiết bị mạng cũng chạy phần mềm và có thể chứa lỗ hổng Buffer Overflow — Sysadmin cần cập nhật bản vá và triển khai IPS dù không trực tiếp sửa code (mục 5) |
| Chỉ cần cài Antivirus là đủ chống mọi loại Malware | Đánh giá quá cao khả năng của 1 lớp phòng thủ duy nhất | Antivirus chủ yếu hiệu quả với Virus/1 số Trojan đã có signature; Worm lây lan qua lỗ hổng mạng cần patch management, Ransomware cần thêm backup/segmentation (mục 6) |
| Nhân viên bị lừa qua Social Engineering là do thiếu hiểu biết công nghệ | Đánh giá sai bản chất của Social Engineering | Social Engineering khai thác tâm lý (tin tưởng, sợ hãi, tò mò) chứ không phải thiếu kiến thức kỹ thuật — người am hiểu công nghệ vẫn có thể bị lừa nếu kịch bản đủ tinh vi (mục 8) |

## 17. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Phân biệt DoS và DDoS.**
→ DoS xuất phát từ 1 nguồn duy nhất gửi lượng lớn traffic/request vượt khả năng xử lý mục tiêu. DDoS dùng nhiều nguồn phân tán (thường là botnet gồm hàng nghìn thiết bị bị chiếm quyền) cùng tấn công 1 mục tiêu, khó phòng chống hơn vì không thể chỉ chặn 1 vài địa chỉ IP cụ thể.

**2. Man-in-the-Middle Attack là gì, và ARP Spoofing liên quan thế nào tới loại tấn công này?**
→ MITM là kẻ tấn công chen vào giữa 2 bên đang giao tiếp để nghe lén hoặc chỉnh sửa traffic mà cả 2 bên không biết. ARP Spoofing là 1 trong những vector phổ biến nhất để thực hiện MITM ở Lớp 2 — đánh lừa 2 thiết bị gửi traffic qua đúng máy kẻ tấn công trước khi tới đích thật.

**3. Vì sao Credential Stuffing vẫn thành công dù nạn nhân đã dùng mật khẩu đủ dài và phức tạp?**
→ Vì Credential Stuffing dùng lại chính xác cặp username/password đã bị lộ từ 1 hệ thống khác — độ dài/độ phức tạp của mật khẩu không có ý nghĩa gì khi kẻ tấn công đã có sẵn giá trị đúng. Biện pháp phòng chống hiệu quả là MFA và không tái sử dụng mật khẩu giữa các hệ thống.

**4. Phân biệt Port Security và DAI trong việc chống lại Spoofing.**
→ Port Security kiểm soát địa chỉ MAC được phép trên từng port, chống MAC Address Spoofing/MAC Flooding nhưng không xác thực danh tính thật. DAI kiểm tra ánh xạ IP-MAC trong gói ARP đối chiếu Binding Table, chống ARP Spoofing — 2 cơ chế bổ sung cho nhau chứ không thay thế nhau.

**5. Nêu 1 ví dụ Social Engineering không dùng email/Internet, và giải thích cơ chế.**
→ Tailgating — kẻ tấn công bám theo 1 nhân viên có thẻ ra vào hợp lệ để lẻn vào khu vực vật lý được kiểm soát mà không cần thẻ riêng, khai thác tâm lý ngại từ chối/nghi ngờ người đi cùng của nạn nhân.

## 18. Tài liệu tham khảo

- [Cisco — Port Security Configuration Guide (Catalyst 9300)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-9/configuration_guide/sec/b_179_sec_9300_cg/port_security.html)
- [Cisco — Configuring Dynamic ARP Inspection](https://www.cisco.com/en/US/docs/switches/lan/catalyst3850/software/release/3.2_0_se/multibook/configuration_guide/b_consolidated_config_guide_3850_chapter_0110111.pdf)
- [Cisco — Enhance STP with Root Guard](https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/10588-74.html)
- [Fortinet — Buffer Overflow Attacks, Types & Vulnerabilities](https://www.fortinet.com/resources/cyberglossary/buffer-overflow)
- [CrowdStrike — 12 Types of Malware + Examples](https://www.crowdstrike.com/en-us/cybersecurity-101/malware/types-of-malware/)
- [CrowdStrike — 10 Types of Social Engineering Attacks](https://www.crowdstrike.com/en-us/cybersecurity-101/social-engineering/types-of-social-engineering-attacks/)

---
**Liên kết**: [[Bai-25_Cisco-Wireless-Controller|← Bài 25]] · [[Bai-27_AAA|Bài 27 - AAA →]]
