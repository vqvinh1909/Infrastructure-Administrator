---
title: Bài 17 - Advanced ACL (Extended ACL)
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — Extended ACL là công cụ lọc traffic chi tiết nhất trên router/switch, thường được test trực tiếp trong phỏng vấn thực hành bằng 1 mô hình yêu cầu cụ thể"
tags: [networking, cisco, acl, extended-acl, security, sysadmin]
aliases: [Extended ACL, ACL nâng cao]
related: ["[[Bai-16_Basic-ACL]]", "[[Bai-20_NAT]]"]
last_review: 2026-07-21
---

# Bài 17: Advanced ACL (Extended ACL)

> [!important] Tại sao cần file này
> [[Bai-16_Basic-ACL|Bài 16]] đã chỉ ra hạn chế lớn nhất của Standard ACL: chỉ lọc được theo địa chỉ nguồn, không phân biệt được đích, giao thức hay port — nghĩa là không thể diễn đạt các yêu cầu thực tế kiểu "chỉ cho phép PC A truy cập Server B qua HTTPS, chặn mọi thứ khác". **Extended ACL** giải quyết đúng vấn đề đó, và là dạng ACL được dùng phổ biến nhất trong vận hành thực tế vì độ chi tiết nó mang lại.

## 1. Extended ACL — cơ chế lọc gói tin

Extended ACL lọc dựa trên **nhiều tiêu chí cùng lúc**: địa chỉ nguồn, địa chỉ đích, giao thức (TCP/UDP/ICMP...), và port (nếu là TCP/UDP). Cú pháp cơ bản:

```
Router(config)# access-list <số hiệu> {permit|deny} <giao thức> <nguồn> <wildcard nguồn> [eq <port nguồn>] <đích> <wildcard đích> [eq <port đích>]
```

Ví dụ: chỉ cho phép mạng IT (`10.10.20.0/24`) truy cập Server (`20.20.20.10`) qua HTTPS (port 443), chặn mọi thứ khác tới server đó:

```
Router(config)# access-list 110 permit tcp 10.10.20.0 0.0.0.255 host 20.20.20.10 eq 443
Router(config)# access-list 110 deny ip any host 20.20.20.10
Router(config)# access-list 110 permit ip any any
```

> [!question] Vì sao dòng thứ 2 (`deny ip any host 20.20.20.10`) vẫn cần thiết dù dòng 1 đã giới hạn theo IT + port 443?
> Vì ACL xét tuần tự từng dòng — nếu không có dòng `deny` tường minh này, traffic từ nguồn **khác** IT hoặc **khác** port 443 đi tới server đó sẽ **không khớp dòng 1**, rồi rơi xuống dòng 3 (`permit ip any any`) và **vẫn được cho qua** vì dòng 3 cho phép mọi thứ khác. Dòng `deny` ở giữa đảm bảo mọi traffic khác tới đúng server đó (ngoài trường hợp đã permit ở dòng 1) đều bị chặn, trong khi traffic tới các đích **khác** server đó vẫn được xử lý bình thường ở dòng 3.

`host 20.20.20.10` là cách viết tắt của `20.20.20.10 0.0.0.0` (wildcard toàn số 0 nghĩa là phải khớp chính xác từng bit).

## 2. Vị trí đặt và gán trên Interface

```
Bai 17 - Extended ACL dat gan NGUON:
+--------------+  +----------------+  +--------+  +--------------+
|     PC-A     |--|       R1       |--|   R2   |--|    SERVER    |
|  10.10.10.5  |  | (Extended ACL  |  |        |  |  20.20.20.5  |
|              |  |   dat o day)   |  |        |  |              |
+--------------+  +----------------+  +--------+  +--------------+
```

Khác với Standard ACL (đặt gần đích, [[Bai-16_Basic-ACL|Bài 16]] mục 3), **Extended ACL nên đặt càng gần nguồn càng tốt** — vì nó đã biết đủ thông tin (nguồn, đích, port) để lọc chính xác ngay từ đầu, không cần để traffic "không mong muốn" đi xa vào trong mạng rồi mới bị chặn, tiết kiệm băng thông và tài nguyên xử lý trên các thiết bị trung gian.

```
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ip access-group 110 in
```

> [!question] Vì sao nguyên tắc vị trí đặt của Standard ACL và Extended ACL lại ngược nhau?
> Vì bản chất khác biệt về lượng thông tin mỗi loại có: Standard ACL chỉ biết nguồn nên phải đợi tới gần đích mới lọc an toàn (tránh chặn nhầm traffic hợp lệ đi tới đích khác). Extended ACL biết cả nguồn lẫn đích nên có thể lọc chính xác **ngay tại nguồn** mà không sợ chặn nhầm, đồng thời mang lại lợi ích phụ là ngăn traffic không cần thiết chiếm băng thông đi xuyên qua toàn bộ mạng trước khi bị loại bỏ.

## 3. Numbered và Named Extended ACL

| Kiểu | Dải số/cú pháp |
|---|---|
| **Extended Numbered ACL** | `100-199`, `2000-2699` |
| **Extended Named ACL** | `ip access-list extended <tên>` |

```
Router(config)# ip access-list extended IT-TO-SERVER-HTTPS
Router(config-ext-nacl)# permit tcp 10.10.20.0 0.0.0.255 host 20.20.20.10 eq 443
Router(config-ext-nacl)# deny ip any host 20.20.20.10
Router(config-ext-nacl)# permit ip any any
```

Chèn thêm 1 dòng vào giữa bằng sequence number, ví dụ cho phép thêm SSH quản trị từ 1 IP admin cụ thể:

```
Router(config)# ip access-list extended IT-TO-SERVER-HTTPS
Router(config-ext-nacl)# 5 permit tcp host 10.10.20.99 host 20.20.20.10 eq 22
```

Sequence `5` được chèn **trước** dòng gốc (sequence 10 mặc định) — đảm bảo quy tắc SSH riêng cho admin được xét trước quy tắc HTTPS chung.

## 4. Từ khóa `established` — lọc traffic phản hồi TCP

Extended ACL có thể dùng từ khóa `established` để chỉ cho phép các gói TCP là **phần phản hồi của 1 kết nối đã được khởi tạo từ bên trong**, không cho phép gói tin khởi tạo kết nối mới từ bên ngoài:

```
Router(config)# access-list 110 permit tcp any host 20.20.20.10 established
```

> [!question] `established` hoạt động dựa trên cơ chế nào, và vì sao nó không biến ACL thành "stateful" thực sự?
> `established` chỉ đơn giản kiểm tra xem gói TCP có cờ **ACK hoặc RST** được bật hay không (đặc trưng của gói phản hồi trong bắt tay TCP) — nó **không** thực sự theo dõi trạng thái phiên (session state) như firewall stateful thực thụ (sẽ học khái niệm Stateless/Stateful đầy đủ ở Bài 28). Đây là lý do `established` vẫn có thể bị giả mạo: 1 kẻ tấn công có thể cố tình gửi gói TCP với cờ ACK bật sẵn để "giả vờ" là traffic phản hồi hợp lệ và lách qua ACL — ACL không đủ thông minh để phát hiện gói đó không thuộc về bất kỳ phiên nào đã thực sự được khởi tạo.

## 5. Cấu hình cho mô hình cụ thể

Mô hình: cho phép toàn bộ mạng nội bộ (`10.10.0.0/16`) ra Internet qua HTTP/HTTPS, chặn các giao thức khác (lưu ý: ACL mẫu này chưa có dòng `permit udp 10.10.0.0 0.0.255.255 any eq 53` cho DNS nên máy nội bộ chưa phân giải được tên miền; thực tế cần thêm dòng này trước `deny ip any any`), đồng thời cho phép ICMP để hỗ trợ troubleshoot:

```
Router(config)# ip access-list extended OUT-TO-INTERNET
Router(config-ext-nacl)# permit tcp 10.10.0.0 0.0.255.255 any eq 80
Router(config-ext-nacl)# permit tcp 10.10.0.0 0.0.255.255 any eq 443
Router(config-ext-nacl)# permit icmp 10.10.0.0 0.0.255.255 any
Router(config-ext-nacl)# deny ip any any
Router(config-ext-nacl)# exit
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# ip access-group OUT-TO-INTERNET in
```

## 6. Kiểm tra, giám sát, quản lý

```
Router# show access-lists OUT-TO-INTERNET
Router# show ip interface gigabitEthernet 0/0
Router# show access-lists | include 110
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Traffic hợp lệ (đúng nguồn/đích/port) vẫn bị chặn | Thứ tự dòng ACL sai — 1 dòng `deny` phía trên khớp trước dòng `permit` mong muốn (ACL xét tuần tự, dừng ngay khi khớp dòng đầu tiên) | `show access-lists` xem thứ tự và hit count từng dòng, xác định dòng nào đang thực sự khớp trước |
| Ứng dụng TCP kết nối được nhưng không nhận phản hồi | Thiếu `established` hoặc thiếu dòng permit cho traffic chiều ngược lại ở ACL đặt trên interface đối diện | Kiểm tra ACL trên cả 2 chiều interface liên quan, đảm bảo traffic phản hồi có đường về |
| ACL dùng `established` nhưng vẫn có traffic lạ lọt qua | Kẻ tấn công giả mạo cờ ACK/RST để lách qua (mục 4) — hạn chế cố hữu của ACL stateless | Cân nhắc dùng firewall stateful (Bài 28) cho các vị trí cần bảo mật cao hơn Extended ACL có thể cung cấp |
| ACL không lọc đúng do thứ tự nguồn/đích trong câu lệnh bị đảo | Extended ACL gõ nhầm thứ tự (đích trước, nguồn sau) so với cú pháp chuẩn `<nguồn> <đích>` | Đối chiếu lại cú pháp `access-list ... permit <protocol> <nguồn> <wildcard> <đích> <wildcard>`, không được đảo thứ tự |

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 2 router, 1 PC nguồn, 1 Server đích với 2 dịch vụ giả lập (HTTP, SSH).

1. Cấu hình Extended Named ACL theo mục 3, chỉ cho phép PC nguồn truy cập Server qua HTTPS (443), chặn mọi thứ khác tới server đó.
2. Gán ACL đúng chiều `in`, gần nguồn nhất có thể theo mục 2, xác nhận HTTPS hoạt động, các dịch vụ khác (ví dụ Telnet/SSH) tới đúng server bị chặn.
3. Chèn thêm 1 dòng permit SSH riêng cho 1 IP admin bằng sequence number theo mục 3, xác nhận chỉ đúng IP đó truy cập SSH được, các IP khác vẫn bị chặn.
4. Thử thêm `established` cho 1 dòng ACL theo mục 4, quan sát hành vi khi PC nguồn chủ động kết nối ra ngoài so với khi có thiết bị khác cố kết nối vào.

**Kết quả kỳ vọng**: thành thạo cấu hình lọc traffic chi tiết theo nguồn-đích-port, hiểu rõ vì sao thứ tự các dòng ACL quan trọng, và nắm được giới hạn thực sự của `established`.

## 8. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Thứ tự các dòng trong ACL không quan trọng, miễn nội dung đúng | Bỏ qua cơ chế xét tuần tự, dừng ngay khi khớp dòng đầu tiên | ACL luôn xét từ trên xuống, dừng lại ngay khi gặp dòng đầu tiên khớp — thứ tự sai có thể khiến 1 dòng `permit`/`deny` phía dưới không bao giờ được áp dụng |
| `established` biến ACL thành tường lửa stateful thực sự | Nhầm việc kiểm tra cờ TCP với việc theo dõi trạng thái phiên đầy đủ | `established` chỉ kiểm tra cờ ACK/RST, không theo dõi state thật, vẫn có thể bị giả mạo (mục 4) |
| Extended ACL luôn nên đặt gần đích giống Standard ACL | Áp dụng máy móc quy tắc của Standard ACL sang Extended ACL | Extended ACL nên đặt gần nguồn vì đã đủ thông tin lọc chính xác ngay từ đầu (mục 2) |
| `host 20.20.20.10` và `20.20.20.10 0.0.0.0` là 2 cách viết khác nhau về ý nghĩa | Nhầm lẫn cú pháp rút gọn với cú pháp khác biệt | `host` chỉ là cách viết tắt, hoàn toàn tương đương wildcard `0.0.0.0` (khớp chính xác 1 địa chỉ) |

## 9. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Extended ACL khác Standard ACL ở điểm cốt lõi nào?**
→ Extended ACL lọc được theo nhiều tiêu chí cùng lúc: địa chỉ nguồn, địa chỉ đích, giao thức, và port — trong khi Standard ACL chỉ lọc được theo địa chỉ nguồn duy nhất.

**2. Vì sao Extended ACL nên đặt gần nguồn còn Standard ACL nên đặt gần đích?**
→ Extended ACL đã biết đủ thông tin (nguồn, đích, port) để lọc chính xác ngay tại nguồn mà không sợ chặn nhầm, đồng thời tránh lãng phí băng thông cho traffic sẽ bị chặn đi xuyên qua mạng. Standard ACL chỉ biết nguồn nên phải đợi gần đích mới lọc an toàn, tránh chặn nhầm traffic hợp lệ tới các đích khác.

**3. Từ khóa `established` trong Extended ACL hoạt động dựa trên cơ chế gì, và có hạn chế gì?**
→ Kiểm tra cờ ACK hoặc RST trong TCP header để nhận diện gói phản hồi của 1 kết nối đã khởi tạo. Hạn chế: không thực sự theo dõi trạng thái phiên như firewall stateful, có thể bị giả mạo bằng cách cố tình bật cờ ACK trong gói tấn công.

**4. Vì sao thứ tự các dòng trong 1 ACL lại quan trọng?**
→ Vì router xét ACL tuần tự từ trên xuống và dừng lại ngay khi gói tin khớp dòng đầu tiên — nếu 1 dòng `deny` rộng đặt trước 1 dòng `permit` cụ thể hơn, dòng permit đó sẽ không bao giờ được áp dụng dù cú pháp hoàn toàn đúng.

**5. Vì sao vẫn cần 1 dòng `deny` tường minh giữa ACL dù đã có implicit deny ở cuối, trong ví dụ chặn truy cập 1 server cụ thể?**
→ Vì implicit deny chỉ chặn traffic không khớp **bất kỳ dòng nào**. Nếu ACL có thêm 1 dòng `permit ip any any` ở cuối (cho phép các đích khác), traffic không khớp dòng permit đầu (do sai nguồn/port) nhưng vẫn nhắm tới đúng server đó sẽ "lọt" xuống dòng permit any any nếu không có dòng deny tường minh chặn riêng traffic tới server đó ở giữa.

## 10. Tài liệu tham khảo

- [Cisco — Configure IP Access Lists](https://www.cisco.com/c/en/us/support/docs/security/ios-firewall/23602-confaccesslists.html)
- [ComputerNetworkingNotes — The ip access-list command options and arguments](https://www.computernetworkingnotes.com/ccna-study-guide/the-ip-access-list-command-options-and-arguments.html)
- [Cisco — IP Access List Entry Sequence Numbering](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/xe-16-12/sec-data-acl-xe-16-12-book/sec-acl-seq-num.html)

---
**Liên kết**: [[Bai-16_Basic-ACL|← Bài 16]] · [[Bai-18_NTP-va-Syslog|Bài 18 - NTP và Syslog →]]
