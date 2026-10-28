---
title: Bài 16 - Basic ACL (Standard ACL)
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — ACL là kỹ năng lọc traffic nền tảng, xuất hiện trong hầu như mọi bài test thực hành lẫn câu hỏi phỏng vấn về bảo mật hạ tầng"
tags: [networking, cisco, acl, standard-acl, security, sysadmin]
aliases: [ACL, Access Control List, Standard ACL]
related: ["[[Bai-11_Gioi-thieu-ve-Dinh-tuyen]]", "[[Bai-17_Advanced-ACL]]"]
last_review: 2026-07-21
---

# Bài 16: Basic ACL (Standard ACL)

> [!important] Tại sao cần file này
> Module 3 (Bài 11-15) tập trung vào việc router **chuyển tiếp** gói tin đúng đường. Nhưng "đúng đường" không có nghĩa là "được phép đi" — trong hạ tầng doanh nghiệp thực tế, không phải mọi traffic hợp lệ về mặt định tuyến đều nên được cho qua (ví dụ traffic từ mạng Khách/Guest không nên chạm được vào mạng Kế toán). **ACL (Access Control List)** là công cụ đầu tiên và cơ bản nhất để kiểm soát điều đó ngay trên router/switch, trước khi tới các lớp bảo mật chuyên sâu hơn ở Module 6.

## 1. Ý nghĩa và chức năng ACL

**ACL (Access Control List)** là tập hợp các câu lệnh `permit`/`deny` được router/switch xét tuần tự để quyết định gói tin nào được đi qua, gói tin nào bị chặn. ACL có 2 chức năng chính trong hạ tầng doanh nghiệp:

- **Lọc traffic (filtering)**: chặn hoặc cho phép traffic dựa trên tiêu chí cụ thể — đây là chức năng bảo mật cốt lõi, dùng thay thế hoặc bổ sung cho firewall chuyên dụng ở những vị trí không cần thiết bị riêng.
- **Xác định traffic cho các tính năng khác**: nhiều tính năng Cisco IOS (route filtering, NAT ở [[Bai-20_NAT|Bài 20]], QoS, VPN ở Bài 22) dùng ACL để "chỉ định" traffic nào áp dụng tính năng đó, không trực tiếp mang nghĩa permit/deny.

> [!question] ACL trên router có thay thế được firewall chuyên dụng không?
> Không hoàn toàn. ACL trên router/switch là **stateless** (không theo dõi trạng thái kết nối, xét từng gói tin độc lập — chi tiết khái niệm Stateless sẽ học ở Bài 28), trong khi firewall chuyên dụng thường **stateful** (theo dõi trạng thái phiên, thông minh hơn nhiều). ACL phù hợp làm lớp lọc cơ bản, nhanh, ngay tại router — nhưng với nhu cầu bảo mật phức tạp (kiểm tra sâu nội dung gói tin, chống tấn công nâng cao), vẫn cần thiết bị firewall chuyên dụng như ASA sẽ học ở Bài 28.

## 2. Standard ACL — cơ chế lọc gói tin

**Standard ACL** chỉ lọc dựa trên **duy nhất 1 tiêu chí: địa chỉ IP nguồn (source)** — không xét được địa chỉ đích, port, hay giao thức. Cú pháp cơ bản:

```
Router(config)# access-list <số hiệu> {permit|deny} <địa chỉ nguồn> <wildcard mask>
```

Ví dụ: chặn toàn bộ traffic từ mạng `10.10.30.0/24` (ví dụ mạng Khách/Guest):

```
Router(config)# access-list 10 deny 10.10.30.0 0.0.0.255
Router(config)# access-list 10 permit any
```

> [!question] Vì sao dòng `permit any` ở cuối luôn cần thiết, dù không ai yêu cầu?
> Vì mọi ACL Cisco IOS đều có 1 dòng **"deny any" ẩn (implicit deny)** ở cuối cùng mà quản trị viên không nhìn thấy trong cấu hình — nếu gói tin không khớp bất kỳ dòng `permit`/`deny` tường minh nào ở trên, nó sẽ **tự động bị chặn**. Nếu quên `permit any` ở cuối trong ví dụ trên, ACL sẽ vô tình chặn **toàn bộ traffic khác**, không chỉ riêng mạng Khách — đây là lỗi thực chiến cực kỳ phổ biến và nguy hiểm, có thể gây mất kết nối toàn bộ mạng ngay khi gán ACL vào interface.

Wildcard mask (đã học ở [[Bai-05_Dia-chi-IPv4|Bài 5]]) dùng để xác định phạm vi địa chỉ cần so khớp: bit `0` nghĩa là **phải khớp chính xác**, bit `1` nghĩa là **bỏ qua, không cần khớp**.

## 3. Vị trí đặt và gán ACL trên Interface

Vì Standard ACL chỉ biết được địa chỉ **nguồn** (không biết đích), nếu đặt ACL quá gần nguồn, nó có thể vô tình chặn nhầm traffic hợp lệ dự định đi tới các đích khác mà quản trị viên không có ý định chặn. Do đó, nguyên tắc chuẩn:

```
Bai 16 - Standard ACL dat gan DICH:
+--------------+  +--------+  +----------------+  +--------------+
|     PC-A     |--|   R1   |--|       R2       |--|    SERVER    |
|  10.10.10.5  |  |        |  | (Standard ACL  |  |  20.20.20.5  |
|              |  |        |  |   dat o day)   |  |              |
+--------------+  +--------+  +----------------+  +--------------+
```

**Standard ACL nên đặt càng gần đích (destination) càng tốt** — vì nó chỉ lọc theo nguồn, đặt gần đích tránh chặn nhầm traffic từ cùng nguồn đó nhưng đang đi tới các đích hợp lệ khác trên đường đi.

Gán ACL vào interface theo chiều `in` (traffic đi vào interface đó) hoặc `out` (traffic đi ra):

```
Router(config)# interface gigabitEthernet 0/1
Router(config-if)# ip access-group 10 out
```

> [!question] Vì sao ACL 10 ở ví dụ trên gán theo chiều `out` trên interface hướng tới SERVER, không phải `in`?
> Vì mục tiêu là chặn traffic **trước khi nó tới được SERVER**, và interface `gigabitEthernet 0/1` ở đây là interface **hướng ra phía SERVER** trên R2. Chiều `out` nghĩa là ACL kiểm tra traffic **ngay trước khi nó rời khỏi** interface đó để đi tới đích — đúng vị trí cần chặn theo nguyên tắc "đặt gần đích" ở trên. Nếu gán nhầm `in` trên interface hướng vào PC-A, ACL sẽ áp dụng sai vị trí và có thể chặn nhầm nhiều traffic khác đi qua router đó.

## 4. Numbered ACL và Named ACL

| Kiểu | Dải số/cú pháp | Đặc điểm |
|---|---|---|
| **Standard Numbered ACL** | `1-99`, `1300-1999` | Nhận diện bằng số, không thể chèn/xóa 1 dòng cụ thể (Cisco IOS cũ, trước 12.3(2)T) mà không xóa toàn bộ và gõ lại; từ IOS 12.3(2)T, numbered ACL cũng hỗ trợ sequence number khi sửa bằng `ip access-list standard <số>` |
| **Standard Named ACL** | `ip access-list standard <tên>` | Nhận diện bằng tên gợi nhớ, hỗ trợ **sequence number** để chèn/sửa/xóa từng dòng riêng lẻ mà không ảnh hưởng các dòng khác |

**Named ACL — triển khai và edit:**

```
Router(config)# ip access-list standard CHAN-GUEST
Router(config-std-nacl)# deny 10.10.30.0 0.0.0.255
Router(config-std-nacl)# permit any
```

Mỗi dòng tự động được gán 1 **sequence number** (mặc định tăng dần 10, 20, 30...). Muốn chèn 1 dòng mới vào giữa, chỉ cần định trước, không phải xóa toàn bộ ACL:

```
Router(config)# ip access-list standard CHAN-GUEST
Router(config-std-nacl)# 15 deny 10.10.31.0 0.0.0.255
```

Muốn xóa đúng 1 dòng theo sequence number:

```
Router(config-std-nacl)# no 15
```

> [!question] Vì sao Named ACL với sequence number được khuyến nghị hơn hẳn Numbered ACL trong vận hành thực tế?
> Vì Numbered ACL (đặc biệt trên IOS cũ, trước 12.3(2)T; từ 12.3(2)T numbered ACL cũng sửa được theo sequence number qua `ip access-list standard|extended <số>`) không cho phép sửa/chèn 1 dòng riêng lẻ — muốn thêm 1 điều kiện mới vào giữa danh sách, phải `no access-list <số>` để xóa **toàn bộ** ACL đó rồi gõ lại từ đầu, rất rủi ro nếu ACL đang áp dụng cho interface đang có traffic thật (dù chỉ mất vài giây "trống ACL" cũng đủ gây ra khoảng hở bảo mật hoặc ngắt kết nối tạm thời). Named ACL với sequence number giải quyết đúng vấn đề này — chèn/xóa từng dòng độc lập, an toàn hơn nhiều khi thao tác trên hệ thống đang chạy (tính năng Sequence Number cũng áp dụng được cho cả Named ACL dạng cũ nếu IOS đủ mới).

## 5. Cấu hình cho mô hình cụ thể

Mô hình: chỉ cho phép mạng IT (`10.10.20.0/24`) truy cập vào Server quản trị (`20.20.20.10`), chặn mọi mạng khác:

```
Router(config)# ip access-list standard ONLY-IT-TO-SERVER
Router(config-std-nacl)# permit 10.10.20.0 0.0.0.255
Router(config-std-nacl)# exit
Router(config)# interface gigabitEthernet 0/1
Router(config-if)# ip access-group ONLY-IT-TO-SERVER out
```

Ở đây không cần dòng `deny any` tường minh vì **implicit deny** (mục 2) đã tự động chặn mọi nguồn khác ngoài mạng IT.

## 6. Kiểm tra, giám sát, quản lý ACL

```
Router# show access-lists
Router# show access-lists ONLY-IT-TO-SERVER
Router# show ip interface gigabitEthernet 0/1
```

`show access-lists` hiển thị toàn bộ ACL cùng **số lượng match (hit count)** cho từng dòng — cực kỳ hữu ích để xác nhận ACL có thực sự "bắt" đúng traffic dự kiến hay không, hoặc phát hiện dòng nào không bao giờ khớp (có thể ACL đặt sai vị trí hoặc sai chiều). `show ip interface <tên>` cho biết ACL nào đang được gán theo chiều `in`/`out` trên interface đó.

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Toàn bộ mạng mất kết nối ngay sau khi gán ACL vào interface | Quên dòng `permit any` cuối cùng, implicit deny chặn hết mọi traffic khác (mục 2) | Thêm `permit any` ở cuối ACL (nếu dùng Named ACL với sequence number, có thể thêm mà không cần xóa lại từ đầu) |
| ACL cấu hình đúng nhưng không lọc được traffic dự kiến | Gán sai chiều (`in` thay vì `out` hoặc ngược lại), hoặc gán sai interface | `show ip interface <tên>` xác nhận đúng ACL, đúng chiều đang áp dụng trên đúng interface |
| Traffic từ nguồn hợp lệ nhưng đi tới đích khác bị chặn nhầm | Standard ACL đặt quá gần nguồn, chặn luôn cả traffic hợp lệ đi tới đích khác (mục 3) | Di chuyển ACL tới gần đích hơn, hoặc chuyển sang Extended ACL ([[Bai-17_Advanced-ACL|Bài 17]]) nếu cần lọc chính xác theo cả nguồn lẫn đích |
| Không xác định được ACL có đang thực sự lọc traffic hay không | Chưa kiểm tra hit count | `show access-lists` xem số lượng match tăng lên theo thời gian thực khi có traffic đi qua |

> [!example] Quan sát ngành — sai sót cấu hình là nguyên nhân hàng đầu của sự cố lọc traffic
> Theo dự báo của Gartner được trích dẫn rộng rãi trong ngành bảo mật, đến năm 2023, **99% sự cố liên quan tới firewall xuất phát từ lỗi cấu hình (misconfiguration), không phải lỗ hổng kỹ thuật của chính thiết bị**. Nguyên lý này áp dụng tương tự cho ACL trên router/switch: lỗi phổ biến nhất không phải do ACL "không đủ mạnh", mà do thao tác cấu hình sai — quên implicit deny, gán sai chiều, đặt sai vị trí như các trường hợp trong bảng trên. Đây là lý do luôn cần kiểm tra `show access-lists` (hit count) và `show ip interface` ngay sau khi thay đổi bất kỳ ACL nào, thay vì chỉ tin tưởng cấu hình đã gõ đúng cú pháp là đủ. Nguồn: [Akamai — The dangers of firewall misconfigurations](https://www.akamai.com/blog/security/the-dangers-of-firewall-misconfigurations-and-how-to-avoid-them).

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer, 1 router, 2 mạng nguồn (IT, Guest), 1 Server đích.

1. Tạo Named Standard ACL chặn mạng Guest, cho phép mọi mạng khác truy cập Server, theo mục 5.
2. Gán ACL đúng chiều `out` trên interface hướng tới Server, xác nhận PC ở mạng IT ping thông Server, PC ở mạng Guest **không** ping thông.
3. Chạy `show access-lists`, quan sát hit count tăng lên tương ứng với traffic vừa test.
4. Cố tình xóa dòng `permit any` (nếu dùng cấu hình có dòng này), quan sát mạng IT cũng mất kết nối tới Server — xác nhận hành vi implicit deny thực tế.
5. Dùng sequence number thêm 1 dòng `deny` mới vào giữa ACL mà không cần xóa lại từ đầu, xác nhận ACL vẫn hoạt động đúng.

**Kết quả kỳ vọng**: hiểu rõ cơ chế lọc theo nguồn của Standard ACL, thành thạo vị trí đặt/gán đúng chiều, và tự tay trải nghiệm hậu quả thực tế của lỗi implicit deny.

## 8. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| ACL không có dòng `deny` tường minh nào thì mặc định cho phép tất cả | Bỏ qua implicit deny ở cuối mọi ACL | Luôn có 1 "deny any" ẩn ở cuối; cần `permit any` tường minh nếu muốn cho phép phần còn lại |
| Standard ACL có thể lọc theo cả nguồn lẫn đích | Nhầm với Extended ACL | Standard ACL chỉ lọc được theo địa chỉ nguồn duy nhất |
| Gán ACL chiều nào (`in`/`out`) cũng cho kết quả như nhau | Bỏ qua sự khác biệt về thời điểm ACL kiểm tra traffic | `in` kiểm tra traffic ngay khi vào interface, `out` kiểm tra ngay trước khi rời interface — chọn sai chiều có thể khiến ACL không lọc đúng traffic dự kiến |
| Numbered ACL và Named ACL khác nhau hoàn toàn về khả năng lọc | Nhầm lẫn giữa cách định danh và khả năng lọc | Cả 2 đều lọc theo cùng cơ chế Standard/Extended; khác biệt chính là cách quản lý/sửa đổi (sequence number) |

## 9. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Standard ACL lọc traffic dựa trên tiêu chí nào? Có hạn chế gì?**
→ Standard ACL chỉ lọc dựa trên địa chỉ IP nguồn duy nhất, không xét được địa chỉ đích, port, hay giao thức — đây là hạn chế lớn nhất so với Extended ACL.

**2. Implicit deny là gì, và vì sao nó nguy hiểm nếu quản trị viên không nắm rõ?**
→ Implicit deny là 1 dòng "deny any" tự động tồn tại ở cuối mọi ACL mà không hiển thị trong cấu hình. Nếu quên thêm `permit any` khi cần cho phép phần traffic còn lại, mọi traffic không khớp các dòng permit/deny tường minh phía trên sẽ bị chặn hết, có thể gây mất kết nối toàn bộ ngoài ý muốn.

**3. Vì sao Standard ACL nên đặt gần đích, còn Extended ACL nên đặt gần nguồn (sẽ học ở Bài 17)?**
→ Vì Standard ACL chỉ biết nguồn, đặt gần đích tránh chặn nhầm traffic từ cùng nguồn đó nhưng đi tới các đích hợp lệ khác. Extended ACL biết cả nguồn lẫn đích nên có thể lọc chính xác ngay từ gần nguồn, tránh traffic không cần thiết đi xa vào mạng trước khi bị chặn.

**4. Named ACL có ưu điểm gì so với Numbered ACL trong vận hành thực tế?**
→ Named ACL hỗ trợ sequence number, cho phép chèn/sửa/xóa từng dòng riêng lẻ mà không cần xóa toàn bộ ACL và gõ lại — an toàn hơn nhiều khi chỉnh sửa ACL đang áp dụng cho hệ thống đang có traffic thật.

**5. Lệnh nào dùng để xác nhận 1 ACL có thực sự đang lọc đúng traffic dự kiến hay không?**
→ `show access-lists` — hiển thị hit count (số lần khớp) cho từng dòng ACL, cho biết dòng nào đang thực sự được traffic thật khớp vào, giúp phát hiện ACL đặt sai vị trí/chiều nếu hit count không như kỳ vọng.

## 10. Tài liệu tham khảo

- [NetworkAcademy.IO — Standard Numbered ACL](https://www.networkacademy.io/ccna/network-security/standard-numbered-acl)
- [Cisco Community — Cisco Access Control Lists (ACL)](https://community.cisco.com/t5/networking-knowledge-base/cisco-access-control-lists-acl/ta-p/4182349)
- [Study CCNA — Configuring Named ACLs](https://study-ccna.com/configuring-named-acls/)
- [Cisco — IP Access List Entry Sequence Numbering](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/xe-16-12/sec-data-acl-xe-16-12-book/sec-acl-seq-num.html)
- [Akamai — The dangers of firewall misconfigurations](https://www.akamai.com/blog/security/the-dangers-of-firewall-misconfigurations-and-how-to-avoid-them)

---
**Liên kết**: [[Bai-15_EIGRP|← Bài 15]] · [[Bai-17_Advanced-ACL|Bài 17 - Advanced ACL →]]
