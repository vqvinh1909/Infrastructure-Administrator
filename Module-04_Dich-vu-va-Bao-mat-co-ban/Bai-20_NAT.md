---
title: Bài 20 - NAT
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — NAT là công nghệ bắt buộc mọi mạng doanh nghiệp đều dùng để ra Internet, và Port Forwarding (Static NAT) là kỹ năng thực chiến publish dịch vụ ra ngoài rất hay bị hỏi"
tags: [networking, cisco, nat, pat, port-forwarding, sysadmin]
aliases: [NAT, Network Address Translation, PAT, Port Forwarding]
related: ["[[Bai-05_Dia-chi-IPv4]]", "[[Bai-19_Dich-vu-DHCP]]"]
last_review: 2026-07-21
---

# Bài 20: NAT (Network Address Translation)

> [!important] Tại sao cần file này
> [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]] và [[Bai-05_Dia-chi-IPv4|Bài 5]] đã nhắc tới sự kiện IANA cạn kiệt địa chỉ IPv4 công cộng (tháng 2/2011) và dải RFC 1918 (IP Private) dùng nội bộ. Nhưng nếu dùng IP Private nội bộ, làm sao thiết bị đó ra được Internet — nơi chỉ định tuyến được bằng IP Public? **NAT (Network Address Translation)** chính là công nghệ "dịch" giữa 2 thế giới địa chỉ đó, và đến nay vẫn là thành phần bắt buộc ở hầu như mọi router biên (edge router) trong doanh nghiệp.

## 1. Ý nghĩa NAT, Inside/Outside, Public/Private

NAT chuyển đổi địa chỉ IP trong header gói tin khi nó đi qua router — cho phép nhiều thiết bị dùng IP Private (không định tuyến được trên Internet) vẫn giao tiếp ra ngoài thông qua 1 hoặc vài IP Public thực sự định tuyến được.

| Thuật ngữ | Ý nghĩa |
|---|---|
| **Inside Local** | IP Private của thiết bị nội bộ, nhìn từ góc độ mạng nội bộ |
| **Inside Global** | IP Public đại diện cho thiết bị nội bộ đó khi ra Internet |
| **Outside Local/Global** | Tương tự nhưng cho thiết bị ở phía ngoài (thường trùng nhau trong đa số mô hình NAT đơn giản) |
| **Inside interface** | Interface hướng vào mạng nội bộ (`ip nat inside`) |
| **Outside interface** | Interface hướng ra Internet (`ip nat outside`) |

## 2. Static NAT, Dynamic NAT, NAT Overload (PAT)

| Loại | Cơ chế | Trường hợp dùng |
|---|---|---|
| **Static NAT** | Ánh xạ **1-1 cố định** giữa 1 IP Private và 1 IP Public, không đổi | Server nội bộ cần luôn truy cập được từ ngoài qua đúng 1 địa chỉ Public (mail server, web server) |
| **Dynamic NAT** | Ánh xạ **nhiều-nhiều** từ 1 pool IP Private sang 1 pool IP Public, cấp theo nguyên tắc ai đến trước dùng trước | Ít phổ biến hơn PAT trong thực tế — hiếm khi có đủ IP Public dư để làm pool 1-1 động cho nhiều thiết bị |
| **NAT Overload (PAT — Port Address Translation)** | Ánh xạ **nhiều-1**: nhiều IP Private cùng dùng chung 1 IP Public, phân biệt bằng **số port** khác nhau | Phổ biến nhất trong thực tế — toàn bộ mạng nội bộ ra Internet chỉ qua 1 IP Public duy nhất |

> [!question] Vì sao PAT có thể cho phép hàng trăm thiết bị cùng dùng chung 1 IP Public mà không bị "lẫn lộn" traffic phản hồi?
> Vì PAT không chỉ đổi địa chỉ IP mà còn đổi luôn **số port nguồn** — mỗi kết nối từ thiết bị nội bộ được gán 1 cặp (IP Public, port) riêng biệt trong bảng NAT. Khi traffic phản hồi từ Internet quay về đúng cặp (IP, port) đó, router tra bảng NAT để biết chính xác cần trả về đúng thiết bị nội bộ nào. Với hơn 65.000 port khả dụng (0-65535) cho mỗi IP Public, PAT có thể phục vụ đồng thời một số lượng lớn kết nối cùng lúc dù chỉ dùng 1 địa chỉ Public duy nhất.

## 3. Cấu hình cho từng loại

**Xác định Inside/Outside interface** (bắt buộc cho mọi loại NAT):

```
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ip nat inside
Router(config)# interface gigabitEthernet 0/1
Router(config-if)# ip nat outside
```

**Static NAT:**

```
Router(config)# ip nat inside source static 10.10.10.10 203.0.113.10
```

**Dynamic NAT:**

```
Router(config)# ip nat pool PUBLIC-POOL 203.0.113.20 203.0.113.30 netmask 255.255.255.0
Router(config)# access-list 1 permit 10.10.10.0 0.0.0.255
Router(config)# ip nat inside source list 1 pool PUBLIC-POOL
```

**NAT Overload (PAT) — dùng chung 1 IP Public của chính interface outside:**

```
Router(config)# access-list 1 permit 10.10.10.0 0.0.0.255
Router(config)# ip nat inside source list 1 interface gigabitEthernet 0/1 overload
```

> [!question] Vì sao PAT ở ví dụ trên dùng `interface gigabitEthernet 0/1` thay vì 1 pool IP cụ thể như Dynamic NAT?
> Vì mục tiêu của PAT thường là dùng **đúng 1 IP Public duy nhất** (thường là IP được ISP cấp cho interface outside) cho toàn bộ mạng nội bộ — dùng thẳng `interface` nghĩa là NAT sẽ tự động lấy địa chỉ IP hiện tại của chính interface đó, không cần khai báo riêng 1 pool. Đây là mô hình phổ biến nhất trong thực tế vì hầu hết doanh nghiệp vừa/nhỏ chỉ được ISP cấp 1 (hoặc rất ít) IP Public.

## 4. Port Forwarding (Static NAT theo Port) — Publish dịch vụ ra ngoài

Khi cần cho phép người dùng từ Internet truy cập **1 dịch vụ cụ thể** (ví dụ web server) chạy trên 1 thiết bị nội bộ, dùng Static NAT kèm theo port:

```
Router(config)# ip nat inside source static tcp 10.10.10.50 80 203.0.113.10 8080
```

Lệnh trên nghĩa là: traffic từ Internet gửi tới `203.0.113.10:8080` sẽ được NAT chuyển tiếp vào đúng `10.10.10.50:80` (web server nội bộ đang lắng nghe port 80) — người dùng ngoài không cần biết địa chỉ IP Private thật của server, chỉ cần biết đúng IP Public + port đã publish.

> [!question] Vì sao Port Forwarding không nhất thiết phải dùng cùng 1 số port ở 2 đầu (ví dụ 8080 ngoài, 80 trong)?
> Vì Port Forwarding về bản chất là ánh xạ **độc lập** giữa cặp (IP, port) bên ngoài và cặp (IP, port) bên trong — không có yêu cầu kỹ thuật nào bắt 2 port phải trùng nhau. Việc cố tình đổi port bên ngoài (ví dụ dùng `8080` thay vì `80`) đôi khi là lựa chọn có chủ đích để "che" bớt thông tin về dịch vụ thật đang chạy port tiêu chuẩn nào, hoặc để cùng 1 IP Public publish được nhiều dịch vụ khác nhau trên các port ngoài khác nhau, dù các dịch vụ đó cùng chạy port 80 ở bên trong trên nhiều server khác nhau.

## 5. Giám sát bảng NAT, xử lý sự cố

```
Router# show ip nat translations
Router# show ip nat statistics
Router# clear ip nat translation *
```

`show ip nat translations` hiển thị toàn bộ các bản ghi dịch địa chỉ hiện có (Inside Local ↔ Inside Global, kèm port nếu là PAT) — lệnh quan trọng nhất để xác nhận NAT có đang hoạt động và dịch đúng địa chỉ hay không. `clear ip nat translation *` xóa toàn bộ bảng NAT hiện tại — hữu ích khi cần buộc router tạo lại bản dịch mới (ví dụ sau khi thay đổi cấu hình NAT).

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Thiết bị nội bộ không ra được Internet dù đã cấu hình NAT | Thiếu `ip nat inside`/`ip nat outside` trên đúng interface, hoặc ACL trong `ip nat inside source list` không khớp đúng dải IP nội bộ | Đối chiếu `show ip interface` xem interface đã đánh dấu inside/outside đúng chưa; kiểm tra lại ACL |
| Port Forwarding cấu hình đúng nhưng người dùng ngoài Internet vẫn không truy cập được dịch vụ | Firewall/ACL ở interface outside chặn port đó, hoặc dịch vụ nội bộ chưa thực sự lắng nghe đúng port | `show ip nat translations` xác nhận bản ghi tồn tại; kiểm tra thêm ACL trên interface outside có cho phép traffic tới đúng port hay không |
| Một số kết nối bị rớt giữa chừng dù NAT vẫn "chạy" | Bảng NAT bị đầy hoặc timeout translation quá ngắn với ứng dụng cần giữ kết nối lâu | `show ip nat statistics` xem số lượng translation hiện tại so với giới hạn, cân nhắc điều chỉnh timeout nếu cần |
| Sau khi đổi cấu hình NAT, hành vi dịch địa chỉ vẫn như cũ | Bảng NAT cũ (translation cache) chưa được xóa, router vẫn dùng bản ghi cũ | `clear ip nat translation *` để buộc tạo lại bản dịch theo cấu hình mới |

## 6. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 1 router biên, 1 mạng nội bộ, 1 "Internet" giả lập, 1 web server nội bộ.

1. Cấu hình PAT (NAT Overload) theo mục 3 cho toàn bộ mạng nội bộ ra "Internet" giả lập, xác nhận PC nội bộ ping thông thiết bị phía Internet.
2. Chạy `show ip nat translations` khi đang có traffic, quan sát các bản ghi PAT với port khác nhau cho cùng 1 IP Public.
3. Cấu hình Port Forwarding theo mục 4 để publish web server nội bộ, xác nhận từ phía "Internet" giả lập truy cập được đúng địa chỉ Public + port đã publish.
4. Thử đổi port ngoài khác với port trong (ví dụ ngoài `8080`, trong `80`), xác nhận vẫn truy cập được bình thường qua port ngoài đã khai báo.

**Kết quả kỳ vọng**: phân biệt rõ 3 loại NAT, thành thạo cấu hình PAT (phổ biến nhất thực tế) và Port Forwarding để publish dịch vụ — kỹ năng thường xuyên dùng khi vận hành hạ tầng doanh nghiệp thực tế.

## 7. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| NAT là 1 tính năng bảo mật thực sự (che giấu mạng nội bộ đồng nghĩa với an toàn) | Nhầm lẫn hiệu ứng phụ (che IP nội bộ) với chức năng bảo mật chủ động | NAT không kiểm tra hay lọc nội dung traffic — vẫn cần ACL ([[Bai-17_Advanced-ACL|Bài 17]]) hoặc firewall (Bài 28) để bảo vệ thực sự |
| Dynamic NAT và PAT là cùng 1 khái niệm | Nhầm lẫn 2 cơ chế nhiều-nhiều và nhiều-một | Dynamic NAT vẫn là ánh xạ 1-1 tại 1 thời điểm (từ pool), PAT mới thực sự cho nhiều thiết bị dùng chung 1 IP nhờ phân biệt bằng port |
| Port Forwarding bắt buộc port ngoài phải trùng port trong | Áp đặt quy tắc không có thật về mặt kỹ thuật | Port ngoài và port trong hoàn toàn độc lập, có thể khác nhau tùy thiết kế (mục 4) |
| Chỉ cần cấu hình `ip nat inside source ...` là đủ, không cần đánh dấu interface | Bỏ qua bước bắt buộc xác định inside/outside | NAT không hoạt động nếu thiếu `ip nat inside`/`ip nat outside` trên đúng interface tương ứng |

## 8. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. NAT giải quyết vấn đề gì trong bối cảnh IPv4 khan hiếm?**
→ Cho phép nhiều thiết bị dùng IP Private (RFC 1918) nội bộ vẫn giao tiếp được ra Internet chỉ qua 1 hoặc vài IP Public, giảm đáng kể nhu cầu IP Public thực tế cần cấp phát cho mỗi tổ chức.

**2. Phân biệt Static NAT, Dynamic NAT và PAT.**
→ Static NAT: ánh xạ 1-1 cố định. Dynamic NAT: ánh xạ nhiều-nhiều từ pool, nhưng vẫn 1-1 tại một thời điểm. PAT (NAT Overload): ánh xạ nhiều-1, phân biệt bằng số port, cho phép nhiều thiết bị dùng chung đúng 1 IP Public.

**3. Vì sao PAT có thể phục vụ hàng trăm thiết bị chỉ với 1 IP Public?**
→ Vì PAT đổi cả IP lẫn số port nguồn, tạo ra hàng nghìn cặp (IP, port) khác nhau có thể dùng cho các kết nối riêng biệt dù chung 1 địa chỉ IP Public, tận dụng không gian port 0-65535 khả dụng.

**4. Port Forwarding (Static NAT theo port) dùng để làm gì, cho ví dụ cụ thể?**
→ Dùng để publish 1 dịch vụ cụ thể chạy trên thiết bị nội bộ ra Internet qua đúng 1 địa chỉ Public + port xác định, ví dụ `ip nat inside source static tcp 10.10.10.50 80 203.0.113.10 8080` cho phép truy cập web server nội bộ (port 80) qua `203.0.113.10:8080` từ ngoài.

**5. NAT có phải là giải pháp bảo mật không? Vì sao?**
→ Không hoàn toàn. NAT có hiệu ứng phụ che giấu địa chỉ IP nội bộ khỏi bên ngoài, nhưng bản thân nó không kiểm tra hay lọc nội dung traffic — vẫn cần ACL hoặc firewall chuyên dụng để có khả năng bảo vệ thực sự.

## 9. Tài liệu tham khảo

- [Cisco — Configuring Static and Dynamic NAT Simultaneously](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13778-9.html)
- [NetworksTraining — How to Configure NAT on Cisco Router Step by Step](https://www.networkstraining.com/configuring-nat-on-cisco-routers/)
- [NetworksTraining — How to Configure Port Forwarding on Cisco Router](https://www.networkstraining.com/cisco-router-port-forwarding-configuration/)
- [Cisco — Configuring NAT and Static PAT to Support an Internal Web Server](https://www.cisco.com/c/en/us/support/docs/long-reach-ethernet-lre-digital-subscriber-line-xdsl/asymmetric-digital-subscriber-line-adsl/12905-827spat.html)

---
**Liên kết**: [[Bai-19_Dich-vu-DHCP|← Bài 19]] · [[Bai-21_FHRP|Bài 21 - FHRP →]]
