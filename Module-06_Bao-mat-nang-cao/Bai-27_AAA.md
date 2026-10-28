---
title: Bài 27 - Cấu hình bảo mật thiết bị (Password, ACL, AAA)
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — bảo mật quyền truy cập vào chính thiết bị mạng (mật khẩu, ACL VTY, AAA/RADIUS/TACACS+) là kỹ năng vận hành cơ bản nhất, luôn xuất hiện trong checklist hardening thiết bị"
tags: [networking, security, password, encryption, acl, vty, aaa, radius, tacacs, 802.1x, sysadmin]
aliases: [Enable Secret, Service Password-Encryption, VTY Access-Class, AAA, RADIUS, TACACS+, 802.1X]
related: ["[[Bai-26_Bao-mat-Lop-2]]", "[[Bai-22_Remote-Access-VPN]]", "[[Bai-16_Basic-ACL]]"]
last_review: 2026-07-21
---

# Bài 27: Cấu hình bảo mật thiết bị (Password, ACL, AAA)

> [!important] Tại sao cần file này
> [[Bai-26_Bao-mat-Lop-2|Bài 26]] bảo vệ switch khỏi thiết bị lạ ở Lớp 2, nhưng chưa trả lời câu hỏi: **ai** được phép đăng nhập vào chính thiết bị mạng (router/switch/firewall) để quản trị? Nếu chính thiết bị quản trị hạ tầng lại dùng mật khẩu yếu, lưu dạng đọc được, hoặc cho phép bất kỳ ai từ bất kỳ đâu Telnet/SSH vào — mọi lớp phòng thủ khác đã học đều trở nên vô nghĩa. Bài này đi từ cấu hình bảo mật cơ bản nhất (mật khẩu, ACL giới hạn truy cập từ xa) tới mô hình quản lý truy cập tập trung (AAA).

## 1. Thiết lập mật khẩu truy cập thiết bị và các dạng mã hóa

Thiết bị Cisco có nhiều "cửa" cần đặt mật khẩu riêng: **console** (truy cập vật lý qua cổng console), **VTY** (Telnet/SSH từ xa), và **enable** (vào chế độ Privileged EXEC — toàn quyền cấu hình). Lưu ý: để `transport input ssh` hoạt động, thiết bị phải có sẵn điều kiện tiên quyết cho SSH — `hostname`, `ip domain-name`, `crypto key generate rsa modulus 2048` — và thực tế thường dùng `login local` cùng `username ... secret ...` thay cho `password`/`login` dùng chung như ví dụ rút gọn dưới đây.

```
Router(config)# line console 0
Router(config-line)# password Console@2026
Router(config-line)# login

Router(config)# line vty 0 4
Router(config-line)# password Vty@2026
Router(config-line)# login
Router(config-line)# transport input ssh

Router(config)# enable secret Enable@2026
```

**Các dạng mã hóa mật khẩu trên Cisco IOS** — khi xem lại `running-config`, mỗi mật khẩu có 1 số hiệu (type) phía trước thể hiện cách nó được lưu:

| Type | Thuật toán | Đánh giá |
|---|---|---|
| **Type 0** | Không mã hóa — hiển thị nguyên văn (plaintext) trong running-config | **Không an toàn**, tuyệt đối không dùng cho môi trường thật |
| **Type 5** | MD5 có salt (hash 1 chiều) | Đã lỗi thời, khuyến nghị chuyển sang Type 8/9 |
| **Type 7** | Thuật toán Vigenère cổ điển | **Đã bị bẻ khóa từ năm 1995**, dễ dàng giải mã ngược bằng công cụ có sẵn — chỉ nên xem là "che mắt" chứ không phải bảo mật thật |
| **Type 8** | PBKDF2 + SHA-256, salt 80-bit, 20.000 vòng lặp | An toàn, khuyến nghị dùng |
| **Type 9** | Thuật toán scrypt | An toàn nhất hiện tại, khuyến nghị dùng |

```
Router(config)# service password-encryption
```

Lệnh `service password-encryption` mã hóa **toàn bộ mật khẩu Type 0** đang có trong cấu hình (như mật khẩu console/VTY vừa khai báo bằng `password`) thành **Type 7** — tốt hơn để trần (Type 0), nhưng vẫn yếu vì Type 7 dễ giải mã ngược. Riêng lệnh `enable secret` (khác với `enable password` đã lỗi thời, không nên dùng) **luôn tự động** mã hóa bằng thuật toán mạnh hơn (Type 8/9 tùy phiên bản IOS) mà không cần thêm `service password-encryption`.

> [!question] Vì sao `enable secret` được khuyến nghị dùng thay vì `enable password`, dù cả 2 đều đặt mật khẩu cho chế độ Privileged EXEC?
> Vì `enable password` chỉ lưu dạng Type 0 (plaintext) hoặc Type 7 (nếu có `service password-encryption`) — cả 2 đều có thể bị đọc ngược lại mật khẩu gốc dễ dàng. `enable secret` dùng thuật toán **hash 1 chiều** (không thể đảo ngược để lấy lại mật khẩu gốc, chỉ có thể so sánh hash) mạnh hơn nhiều (Type 8/9), tương tự nguyên tắc lưu mật khẩu an toàn ở bất kỳ hệ thống nào khác. Nếu vô tình cấu hình cả `enable password` lẫn `enable secret`, IOS sẽ **luôn ưu tiên dùng `enable secret`** và bỏ qua `enable password`.

## 2. Kiểm soát, giới hạn truy cập thiết bị từ xa qua ACL (VTY)

Chỉ đặt mật khẩu VTY (mục 1) là chưa đủ — mật khẩu vẫn có thể bị brute force/dò nếu **bất kỳ ai từ bất kỳ đâu trên Internet** đều gõ được tới đúng cổng SSH/Telnet của thiết bị. Cần giới hạn thêm: chỉ những địa chỉ IP quản trị hợp lệ (ví dụ máy trạm của phòng IT) mới được phép kết nối tới VTY, dùng ACL Standard đã học ở [[Bai-16_Basic-ACL|Bài 16]] áp trực tiếp vào line VTY qua `access-class`.

```
Router(config)# access-list 10 permit 10.10.99.0 0.0.0.255
Router(config)# line vty 0 4
Router(config-line)# access-class 10 in
```

ACL số `10` chỉ cho phép dải mạng `10.10.99.0/24` (giả định là mạng quản trị IT) — mọi địa chỉ IP khác cố gắng SSH/Telnet vào VTY đều bị từ chối kết nối ngay từ bước bắt tay, trước khi kịp thử bất kỳ mật khẩu nào.

> [!question] Vì sao `access-class` áp ACL vào **line VTY** lại khác về bản chất so với `access-group` áp ACL vào **interface** đã học ở Bài 16/17?
> Vì `access-group` (Bài 16/17) lọc traffic **đi qua** interface đó (traffic transit, router chỉ đóng vai trò trung gian chuyển tiếp). `access-class` lọc traffic **đi tới chính thiết bị** (traffic destined-to-device) — cụ thể là ai được phép mở phiên quản trị (Telnet/SSH) vào đúng thiết bị đang cấu hình. Đây là lý do `access-class` luôn áp trong `line configuration mode` (VTY/console) chứ không áp vào interface vật lý — nó bảo vệ **chính thiết bị**, không bảo vệ traffic đi ngang qua thiết bị.

## 3. Mô hình AAA — 3 câu hỏi độc lập

ACL VTY (mục 2) chỉ trả lời "ai được phép **kết nối tới**", còn mật khẩu chung cho mọi người (mục 1) không phân biệt được **từng cá nhân** đã đăng nhập. Khi số lượng người quản trị tăng lên, cần mô hình chi tiết hơn: **AAA**, tách việc kiểm soát truy cập thành 3 câu hỏi riêng biệt, xử lý tuần tự:

| Thành phần | Câu hỏi trả lời | Ví dụ |
|---|---|---|
| **Authentication** (Xác thực) | "Bạn là ai?" | Đăng nhập đúng username/password, hoặc chứng chỉ số |
| **Authorization** (Phân quyền) | "Bạn được làm gì?" | Tài khoản A chỉ được `show`, tài khoản B được toàn quyền cấu hình (`enable`) |
| **Accounting** (Ghi nhận) | "Bạn đã làm gì?" | Ghi log mọi lệnh đã chạy, thời gian đăng nhập/đăng xuất, để phục vụ audit |

> [!question] Vì sao phải tách riêng Authorization khỏi Authentication, thay vì "đăng nhập được là làm được mọi thứ"?
> Vì đây chính là nguyên tắc **Least Privilege** (đặc quyền tối thiểu) — 1 nhân viên IT mới chỉ cần quyền xem log để hỗ trợ, không cần quyền đổi cấu hình router biên. Nếu gộp chung Authentication và Authorization (đăng nhập được là có toàn quyền), rủi ro rất lớn nếu tài khoản đó bị lộ hoặc nhân viên đó vô tình gõ nhầm lệnh nguy hiểm. Tách riêng cho phép quản trị viên cấp đúng quyền cần thiết cho từng vai trò, giảm thiểu thiệt hại nếu 1 tài khoản gặp sự cố.

## 4. Local AAA — Xác thực cục bộ trên thiết bị

Trước khi dùng server tập trung, cách đơn giản nhất là khai báo user cục bộ ngay trên thiết bị — thay thế cho mật khẩu VTY dùng chung ở mục 1 bằng tài khoản **riêng cho từng người**:

```
Router(config)# username admin privilege 15 secret Admin@2026
Router(config)# aaa new-model
Router(config)# aaa authentication login default local
Router(config)# aaa authorization exec default local
```

`aaa new-model` bật cơ chế AAA trên thiết bị (bắt buộc trước mọi cấu hình AAA khác). `aaa authentication login default local` chỉ định: khi đăng nhập, dùng danh sách user **cục bộ** (local) để xác thực — thay vì 1 mật khẩu chung mọi người đều biết như `line vty ... password` ở mục 1. Cách này đơn giản nhưng có nhược điểm: phải lặp lại thủ công trên từng thiết bị, không có điểm quản lý tập trung.

## 5. RADIUS và TACACS+ — Xác thực tập trung qua server

| Tiêu chí | RADIUS | TACACS+ |
|---|---|---|
| Nguồn gốc | Chuẩn mở IETF, hỗ trợ đa hãng | Cisco phát triển ban đầu, nay đã được công bố thành RFC 8907 (2020, Informational) và nhiều hãng hỗ trợ |
| Giao thức truyền tải | UDP (port 1812 xác thực, 1813 accounting) | TCP (port 49) |
| Phạm vi mã hóa | Chỉ mã hóa **mật khẩu** trong gói tin, phần còn lại gửi rõ | Mã hóa **toàn bộ nội dung gói tin** |
| Mô hình AAA | Gộp chung Authentication và Authorization trong cùng 1 phản hồi | Tách riêng biệt Authentication, Authorization, Accounting |
| Trường hợp dùng phổ biến | Xác thực người dùng cuối truy cập mạng (VPN, Wifi, 802.1X) | Quản trị thiết bị mạng (router/switch) — kiểm soát chính xác lệnh nào được phép chạy |

```
Router(config)# radius server RAD-SRV
Router(config-radius-server)# address ipv4 10.10.10.100 auth-port 1812 acct-port 1813
Router(config-radius-server)# key Radius@2026
Router(config)# aaa authentication login default group radius local
```

```
Router(config)# tacacs server TAC-SRV
Router(config-server-tacacs)# address ipv4 10.10.10.101
Router(config-server-tacacs)# key Tacacs@2026
Router(config)# aaa authentication login default group tacacs+ local
```

Cả 2 ví dụ trên đều dùng cú pháp `group radius local`/`group tacacs+ local` — nghĩa là: **ưu tiên hỏi server RADIUS/TACACS+ trước**, nếu server không phản hồi được (mất kết nối) thì **rơi về (fallback) dùng tài khoản local** đã khai báo ở mục 4 — đảm bảo vẫn đăng nhập được vào thiết bị ngay cả khi server AAA tập trung gặp sự cố.

> [!question] Vì sao TACACS+ vẫn là lựa chọn phổ biến để quản trị thiết bị mạng dù RADIUS là chuẩn mở, đa hãng hơn?
> Vì TACACS+ mã hóa **toàn bộ nội dung gói tin** (không chỉ mật khẩu như RADIUS) và tách riêng biệt 3 chức năng AAA — cho phép kiểm soát **chi tiết tới từng lệnh CLI** (ví dụ: tài khoản A chỉ được chạy `show` nhưng không được `configure terminal`). Đây là mức độ kiểm soát rất cần thiết khi quản trị thiết bị hạ tầng mạng, nơi 1 lệnh cấu hình sai có thể gây gián đoạn toàn hệ thống — trong khi RADIUS phù hợp hơn cho xác thực **người dùng cuối** truy cập mạng (VPN, Wifi), nơi yêu cầu chủ yếu chỉ là "cho vào hay không cho vào" chứ không cần kiểm soát từng lệnh.

## 6. 802.1X — Xác thực truy cập theo Port

802.1X là chuẩn IEEE kiểm soát truy cập mạng **ngay tại switchport/AP** — thiết bị chỉ được cấp quyền truy cập mạng thực sự **sau khi** xác thực thành công, khác với Port Security ([[Bai-26_Bao-mat-Lop-2|Bài 26]] mục 10) vốn chỉ kiểm soát dựa trên địa chỉ MAC (dễ giả mạo) chứ không xác thực danh tính thật.

```
802.1X - 3 thanh phan tham gia xac thuc:
+----------------+            +----------------+             +----------------+
|       PC       | --EAPOL--> |     SWITCH     | --RADIUS--> | RADIUS SERVER  |
|  (Supplicant)  |            |(Authenticator) |             | (Auth Server)  |
+----------------+            +----------------+             +----------------+
```

| Vai trò | Thiết bị đảm nhận | Nhiệm vụ |
|---|---|---|
| **Supplicant** | PC/thiết bị đầu cuối | Gửi thông tin xác thực (thường qua phần mềm client tích hợp sẵn trong hệ điều hành) |
| **Authenticator** | Switch/Access Point | Đứng giữa, chuyển tiếp yêu cầu xác thực, **chặn mọi traffic khác** cho tới khi có kết quả |
| **Authentication Server** | Server RADIUS | Kiểm tra thông tin xác thực, quyết định cho phép hay từ chối |

Giao tiếp giữa Supplicant và Authenticator dùng **EAPOL** (EAP over LAN), còn giữa Authenticator và Authentication Server dùng **EAP đóng gói trong RADIUS** — Authenticator đóng vai trò trung gian "chuyển tiếp" chứ không tự quyết định kết quả xác thực.

```
Switch(config)# aaa new-model
Switch(config)# aaa authentication dot1x default group radius
Switch(config)# dot1x system-auth-control
Switch(config)# interface fastEthernet 0/1
Switch(config-if)# authentication port-control auto
Switch(config-if)# dot1x pae authenticator
```

> [!question] Vì sao 802.1X được xem là kiểm soát truy cập mạnh hơn Port Security dù cả 2 đều hoạt động ở switchport?
> Vì Port Security chỉ kiểm tra **địa chỉ MAC** — kẻ tấn công hoàn toàn có thể giả mạo (spoof) MAC address hợp lệ đã biết để vượt qua kiểm tra này (MAC address không phải bí mật, dễ dàng đọc được qua bắt gói tin). 802.1X yêu cầu xác thực bằng **thông tin định danh thật** (username/password, chứng chỉ số) đối chiếu qua Authentication Server tập trung — mạnh hơn nhiều vì kẻ tấn công không thể chỉ "sao chép" 1 địa chỉ MAC để vượt qua, mà cần có thông tin xác thực hợp lệ thực sự.

## 7. Verify & Troubleshoot

```
Router# show running-config | include password|secret
Router# show access-lists 10
Router# show aaa servers
Router# test aaa group radius admin Admin@2026 legacy
Switch# show dot1x interface fastEthernet 0/1
Switch# show authentication sessions
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Mật khẩu vẫn hiển thị dạng đọc được trong `show running-config` | Dùng `password` (Type 0) mà chưa bật `service password-encryption`, hoặc dùng `enable password` thay vì `enable secret` | Bật `service password-encryption`; luôn ưu tiên `enable secret` thay vì `enable password` (mục 1) |
| Không SSH/Telnet được vào thiết bị dù đúng IP nguồn thuộc dải cho phép | ACL số dùng cho `access-class` khai sai dải mạng, hoặc quên `access-class ... in` trên đúng line VTY | `show access-lists 10` đối chiếu đúng dải mạng; kiểm tra đã áp đúng `line vty 0 4` hay chưa (mục 2) |
| Không đăng nhập được vào thiết bị dù đúng tài khoản trên server RADIUS/TACACS+ | Sai `key` chia sẻ giữa thiết bị và server, hoặc server không nhận được request (sai IP/port/firewall chặn) | `show aaa servers` kiểm tra trạng thái kết nối; đối chiếu `key` khớp chính xác 2 đầu |
| Mất kết nối tới server AAA khiến không đăng nhập được vào thiết bị luôn | Thiếu cấu hình fallback `local` trong lệnh `aaa authentication login` | Luôn cấu hình `group radius local`/`group tacacs+ local` để có phương án dự phòng cục bộ |
| PC cắm vào port 802.1X nhưng không có mạng | Port đang ở trạng thái chờ xác thực (`authentication port-control auto` mặc định chặn traffic), Supplicant trên PC chưa cấu hình đúng hoặc xác thực thất bại | `show authentication sessions` xem trạng thái port; kiểm tra cấu hình 802.1X client trên PC |

## 8. Lab thực hành

**Môi trường**: Cisco Packet Tracer (hỗ trợ mô phỏng RADIUS server cơ bản), 1 router, 1 switch, 1 PC.

1. Cấu hình mật khẩu console/VTY/enable theo mục 1, bật `service password-encryption`, xác nhận qua `show running-config` thấy đúng dạng mã hóa tương ứng.
2. Cấu hình ACL giới hạn VTY theo mục 2, thử SSH từ 1 IP ngoài dải cho phép — xác nhận kết nối bị từ chối.
3. Cấu hình Local AAA theo mục 4 trên router, xác nhận đăng nhập bằng tài khoản local thành công.
4. Thêm RADIUS server theo mục 5 (dùng server mô phỏng trong Packet Tracer), cấu hình `aaa authentication login default group radius local`, xác nhận đăng nhập được qua tài khoản trên RADIUS server, rồi cố tình tắt RADIUS server để kiểm chứng fallback về `local`.
5. (Nếu môi trường hỗ trợ) Cấu hình 802.1X cơ bản theo mục 6 trên 1 switchport, quan sát port chặn traffic cho tới khi xác thực thành công.

**Kết quả kỳ vọng**: thành thạo toàn bộ chuỗi bảo mật truy cập thiết bị — từ mật khẩu cơ bản, giới hạn nguồn kết nối bằng ACL, tới quản lý tập trung qua AAA.

## 9. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| `service password-encryption` đủ để bảo vệ mật khẩu an toàn | Đánh giá quá cao độ mạnh của mã hóa Type 7 | Type 7 dùng thuật toán Vigenère đã bị bẻ khóa từ 1995, dễ giải mã ngược bằng công cụ có sẵn — chỉ nên dùng `enable secret` (Type 8/9) cho mật khẩu thực sự quan trọng (mục 1) |
| `access-class` và `access-group` là 2 cách gọi khác nhau của cùng 1 cơ chế | Nhầm lẫn vì cả 2 đều dùng ACL | `access-group` lọc traffic đi qua interface; `access-class` lọc traffic đi tới chính thiết bị qua VTY/console — khác nhau về đối tượng bảo vệ (mục 2) |
| RADIUS và TACACS+ chỉ khác nhau ở cái tên, chức năng giống hệt | Bỏ qua khác biệt về giao thức truyền tải, phạm vi mã hóa, mô hình AAA | RADIUS dùng UDP, chỉ mã hóa mật khẩu, gộp Authentication/Authorization; TACACS+ dùng TCP, mã hóa toàn bộ gói tin, tách riêng 3 thành phần AAA (mục 5) |
| Cấu hình AAA tập trung qua RADIUS/TACACS+ không cần tài khoản local nữa | Bỏ qua rủi ro mất kết nối server AAA | Luôn cần fallback `local` để tránh bị khóa hoàn toàn khỏi thiết bị nếu server AAA tập trung gặp sự cố (mục 5) |
| 802.1X và Port Security là 2 cách gọi khác nhau của cùng 1 cơ chế | Nhầm lẫn vì cả 2 đều hoạt động ở switchport | Port Security chỉ kiểm tra MAC (dễ giả mạo); 802.1X xác thực bằng thông tin định danh thật qua Authentication Server, mạnh hơn nhiều (mục 6) |

## 10. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Vì sao `enable secret` được khuyến nghị dùng thay vì `enable password`?**
→ Vì `enable secret` luôn tự động mã hóa bằng hash 1 chiều mạnh (Type 8/9), không thể đảo ngược để lấy lại mật khẩu gốc — trong khi `enable password` chỉ lưu dạng plaintext hoặc Type 7 (dễ giải mã ngược). Nếu cấu hình cả 2, IOS luôn ưu tiên dùng `enable secret`.

**2. Phân biệt `access-class` và `access-group` khi cùng dùng ACL trên thiết bị Cisco.**
→ `access-group` áp ACL vào interface để lọc traffic **đi qua** thiết bị (transit traffic). `access-class` áp ACL vào line VTY/console để lọc traffic **đi tới chính thiết bị** đó — kiểm soát ai được phép mở phiên quản trị từ xa.

**3. Giải thích 3 thành phần của AAA và mối quan hệ giữa chúng.**
→ Authentication xác định danh tính người dùng ("bạn là ai"). Authorization quyết định quyền hạn sau khi đã xác thực ("bạn được làm gì"). Accounting ghi nhận lại hành động đã thực hiện ("bạn đã làm gì"). Ba bước xử lý tuần tự, Authorization luôn diễn ra sau khi Authentication thành công.

**4. Phân biệt RADIUS và TACACS+ theo giao thức truyền tải và phạm vi mã hóa.**
→ RADIUS dùng UDP (port 1812/1813), chỉ mã hóa phần mật khẩu trong gói tin. TACACS+ dùng TCP (port 49), mã hóa toàn bộ nội dung gói tin, cho phép kiểm soát chi tiết hơn tới từng lệnh CLI.

**5. Vì sao 802.1X được xem là an toàn hơn Port Security dựa trên MAC address?**
→ Vì MAC address có thể bị giả mạo (spoof) dễ dàng, trong khi 802.1X yêu cầu xác thực bằng thông tin định danh thật (tài khoản, chứng chỉ số) đối chiếu qua Authentication Server tập trung, khó bị vượt qua hơn nhiều so với việc chỉ sao chép 1 địa chỉ MAC hợp lệ.

## 11. Tài liệu tham khảo

- [Cisco — Controlling Switch Access with Passwords and Privilege Levels](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-9/configuration_guide/sec/b_179_sec_9300_cg/controlling_switch_access_with_passwords_and_privilege_levels.html)
- [Cisco — Understand Cisco IOS Password Encryption](https://www.cisco.com/c/en/us/support/docs/security-vpn/remote-authentication-dial-user-service-radius/107614-64.html)
- [Cisco — Controlling Access to a Virtual Terminal Line](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/15-mt/sec-data-acl-15-mt-book/sec-cntrl-acc-vtl.pdf)
- [Cisco — Compare TACACS+ and RADIUS](https://www.cisco.com/c/en/us/support/docs/security-vpn/remote-authentication-dial-user-service-radius/13838-10.html)
- [Cisco — 802.1X Port-Based Authentication Configuration Guide](https://www.cisco.com/c/en/us/td/docs/iosxr/cisco8000/security/25xx/configuration/guide/b-system-security-cg-cisco8000-25xx/m-802-1x-authentication-cisco8k.pdf)

---
**Liên kết**: [[Bai-26_Bao-mat-Lop-2|← Bài 26]] · [[Bai-28_Cisco-ASA-Firewall|Bài 28 - Cisco ASA Firewall →]]
