---
title: Bài 7 - Giới thiệu và cấu hình Switch Cisco
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Junior — cấu hình switch cơ bản là bài kiểm tra tay nghề đầu tiên trong hầu hết buổi phỏng vấn thực hành"
tags: [networking, cisco, switch, ios, cli, configuration, sysadmin]
aliases: [Cisco Switch, IOS Configuration, Switch Basics]
related: ["[[Bai-06_Dia-chi-IPv6]]", "[[Bai-04_Thiet-bi-ket-noi-mang]]"]
last_review: 2026-07-21
---

# Bài 7: Giới thiệu và cấu hình Switch Cisco

> [!important] Tại sao cần file này
> Đây là bài đầu tiên chuyển từ "hiểu lý thuyết" sang "tự tay cấu hình thiết bị thật" — đúng khung Understand → Configure → Verify/Troubleshoot của giáo trình. Mọi kỹ thuật ở các bài sau (VLAN, Trunking, STP, định tuyến...) đều được cấu hình trên nền các mode và lệnh cơ bản học ở đây. Một sysadmin không thuộc mode nào đang đứng ở đâu trong CLI sẽ liên tục gõ nhầm lệnh và mất thời gian gấp nhiều lần trong công việc thực tế.

## 1. Các dòng sản phẩm Switch Cisco

| Dòng cũ (đang EOL) | Dòng thay thế hiện tại | Đặc điểm |
|---|---|---|
| Catalyst 2960 | Catalyst 9200 / 9200L | Layer 2, access layer, IOS-XE, hỗ trợ StackWise |
| Catalyst 3560 / 3750 | Catalyst 9200 (bản L3) | Layer 3, có khả năng định tuyến cơ bản |
| Catalyst 3650 / 3850 | Catalyst 9300 / 9300L | Layer 3 hiệu năng cao, uplink module hóa, dùng cho Distribution |

Ký hiệu **"L"** trong tên (ví dụ 9200L, 9300L) nghĩa là **uplink cố định** (không tháo rời được), khác với bản không có "L" cho phép gắn module uplink SFP/SFP+ rời. Dòng 9200/9200L giới hạn PoE ở chuẩn PoE+ (30W), trong khi dòng 9300 hỗ trợ đến PoE++ (mục 7.3, [[Bai-04_Thiet-bi-ket-noi-mang|Bài 4]]). Điểm khác biệt cốt lõi giữa các dòng cũ: 2960 gốc chỉ Layer 2 và không hỗ trợ stack (các bản 2960-S/X/XR sau này có FlexStack/FlexStack-Plus); 3560 có Layer 3 nhưng cũng không stack; 3750 có cả Layer 3 và stack (StackWise) — đây là kiến thức nền để đọc hiểu tài liệu cũ vẫn còn phổ biến trong hạ tầng thực tế tại Việt Nam.

## 2. Thành phần bên ngoài

- **Cổng kết nối (ports)**: RJ45 (UTP), khe SFP/SFP+ cho module quang (mục 7.2, [[Bai-04_Thiet-bi-ket-noi-mang|Bài 4]]).
- **Slot mở rộng**: khe cắm module uplink hoặc module stack (tùy dòng máy).
- **Đèn tín hiệu (LED)**: quan trọng nhất khi troubleshoot tại chỗ — LED **SYST** (trạng thái hệ thống: xanh=OK, hổ phách=lỗi), LED từng cổng thể hiện trạng thái link/tốc độ, và LED **RPS** báo trạng thái nguồn dự phòng (nếu có). Đọc đúng LED giúp loại trừ nhanh sự cố Lớp 1 trước khi mở CLI.

## 3. Thành phần bên trong

| Thành phần | Vai trò | Đặc tính |
|---|---|---|
| **CPU** | Xử lý các tác vụ điều khiển (control plane): chạy giao thức, xử lý lệnh CLI | Không xử lý forwarding tốc độ cao (việc đó do ASIC chuyên dụng đảm nhiệm) |
| **ROM** | Chứa bootstrap program và mã POST (Power-On Self-Test) | Không đổi được, chỉ đọc |
| **RAM** | Lưu running-config và bảng MAC/routing đang hoạt động | **Volatile** — mất hết khi mất điện |
| **NVRAM** | Lưu startup-config | **Non-volatile** — giữ nguyên khi mất điện |
| **Flash** | Lưu file IOS image (và có thể lưu backup cấu hình) | Non-volatile, tương tự ổ đĩa lưu trữ |

**Quy trình khởi động (boot process):**

```
Power On
   |
   v
POST (Power-On Self-Test)          -- chạy từ ROM, kiểm tra phần cứng
   |
   v
Bootstrap program                  -- nạp từ ROM
   |
   v
Nạp IOS image từ FLASH vào RAM     -- IOS bắt đầu chạy
   |
   v
Nạp startup-config từ NVRAM vào RAM -- trở thành running-config
   |
   v
Switch sẵn sàng hoạt động
```

> [!question] Vì sao startup-config lưu ở NVRAM chứ không lưu luôn trong Flash cùng IOS image?
> Về mặt kỹ thuật có thể làm được, nhưng thiết kế tách riêng giúp: (1) NVRAM dung lượng nhỏ, tối ưu cho việc ghi/đọc nhanh một file text nhỏ (vài KB) mỗi khi `write memory`; (2) tách biệt IOS image (ít thay đổi, chỉ đổi khi nâng cấp) khỏi cấu hình (thay đổi thường xuyên) giúp giảm rủi ro hỏng file IOS khi ghi cấu hình liên tục. Đây cũng là lý do lệnh lưu cấu hình (`copy running-config startup-config`) tách biệt hoàn toàn với việc nâng cấp IOS.

## 4. Giới thiệu IOS

**IOS (Internetwork Operating System)** là hệ điều hành chạy trên thiết bị Cisco, cung cấp giao diện dòng lệnh (CLI) để cấu hình và quản trị. Các dòng thiết bị đời mới (Catalyst 9000) chạy **IOS-XE** — phiên bản hiện đại hóa dựa trên nhân Linux, hỗ trợ thêm các tính năng lập trình/tự động hóa (liên quan Bài 29-30) nhưng cú pháp CLI cơ bản gần như không đổi so với IOS truyền thống, nên kiến thức trong bài này áp dụng được cho cả hai.

## 5. Phương thức kết nối quản trị

| Phương thức | Cổng kết nối | Mã hóa | Khi nào dùng |
|---|---|---|---|
| **Console** | Cổng RJ45/USB console riêng, cần cáp console | Không cần (kết nối vật lý trực tiếp) | Cấu hình lần đầu (chưa có IP), hoặc khi mất kết nối mạng cần truy cập khẩn cấp |
| **Telnet** | Qua mạng, cổng TCP 23 | **Không mã hóa** — toàn bộ gõ lệnh và mật khẩu truyền dạng cleartext | Không khuyến nghị dùng trong môi trường thực tế hiện nay |
| **SSH** | Qua mạng, cổng TCP 22 | Có mã hóa | Chuẩn quản trị từ xa hiện nay, bắt buộc trong hạ tầng doanh nghiệp |

> [!example] Case study — Botnet Mirai lây lan qua Telnet mật khẩu mặc định
> Cơ chế lây nhiễm cốt lõi của botnet Mirai (dẫn đến vụ tấn công DDoS vào Dyn DNS đã học ở [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]]) là quét ngẫu nhiên địa chỉ IP tìm cổng Telnet (23/2323) đang mở, sau đó thử brute-force bằng một từ điển khoảng 46-62 cặp username/password mặc định phổ biến (`root/root`, `admin/admin`, `root/xc3511`...). Bất kỳ thiết bị nào để Telnet mở với tài khoản mặc định chưa đổi đều bị chiếm quyền chỉ trong vài giây. Đây là lý do thực chiến cụ thể — không phải nguyên tắc lý thuyết suông — cho việc **luôn tắt Telnet, chỉ dùng SSH, và luôn đổi mật khẩu mặc định** ngay khi đưa thiết bị vào vận hành, dù đó chỉ là switch nội bộ "không lộ ra ngoài Internet trực tiếp". Nguồn: [Heimdal Security — A Technical Analysis of the Mirai Botnet Phenomenon](https://heimdalsecurity.com/blog/mirai-botnet-phenomenon/).

## 6. Các mode cấu hình

```
User EXEC (Switch>)
   | enable
   v
Privileged EXEC (Switch#)
   | configure terminal
   v
Global Configuration (Switch(config)#)
   | interface <tên>  /  vlan <id>  /  line <loại> <phạm vi>
   v
Sub-configuration mode: (config-if)# / (config-vlan)# / (config-line)#
```

| Mode | Dấu nhắc lệnh | Làm được gì | Không làm được gì |
|---|---|---|---|
| User EXEC | `Switch>` | Xem thông tin cơ bản, không đổi được cấu hình | Không xem được running-config, không cấu hình |
| Privileged EXEC | `Switch#` | Xem toàn bộ trạng thái (`show running-config`), sao lưu/phục hồi, reload | Không sửa trực tiếp cấu hình (phải vào Global Config) |
| Global Configuration | `Switch(config)#` | Cấu hình toàn cục: hostname, VLAN, mật khẩu... | Cấu hình riêng từng interface phải vào sub-mode |
| Interface Config | `Switch(config-if)#` | Cấu hình riêng 1 cổng: IP, speed, duplex, VLAN gán vào | — |

Lệnh `exit` lùi lại 1 cấp, `end` (hoặc Ctrl+Z) nhảy thẳng về Privileged EXEC từ bất kỳ sub-mode nào.

## 7. Cấu hình cơ bản ban đầu

```
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW-ACCESS-01
SW-ACCESS-01(config)# enable secret Cisco@2026
SW-ACCESS-01(config)# service password-encryption
SW-ACCESS-01(config)# banner motd #CANH BAO: Chi nguoi duoc uy quyen moi duoc truy cap#
SW-ACCESS-01(config)# no ip domain-lookup
SW-ACCESS-01(config)# line console 0
SW-ACCESS-01(config-line)# password Cisco@Console
SW-ACCESS-01(config-line)# login
SW-ACCESS-01(config-line)# logging synchronous
SW-ACCESS-01(config-line)# exec-timeout 5 0
SW-ACCESS-01(config-line)# exit
SW-ACCESS-01(config)# line vty 0 15
SW-ACCESS-01(config-line)# password Cisco@VTY
SW-ACCESS-01(config-line)# login
SW-ACCESS-01(config-line)# exit
SW-ACCESS-01(config)# end
SW-ACCESS-01# copy running-config startup-config
```

**Giải thích ý nghĩa từng lệnh:**

| Lệnh | Ý nghĩa |
|---|---|
| `enable secret` | Đặt mật khẩu vào Privileged EXEC, tự động **mã hóa** (hash; mặc định type 5/MD5, nên chuyển sang type 8/9 nếu IOS hỗ trợ) — luôn dùng thay vì `enable password` (lưu dạng cleartext) |
| `service password-encryption` | Mã hóa (dạng yếu, type 7, có thể giải mã ngược) toàn bộ mật khẩu dạng cleartext còn lại trong cấu hình (line password) — vẫn nên dùng để tránh lộ mật khẩu khi ai đó nhìn qua màn hình hoặc xem file cấu hình |
| `banner motd` | Thông báo hiển thị trước khi đăng nhập — dùng cho cảnh báo pháp lý (rất quan trọng về mặt bằng chứng nếu xảy ra tranh chấp truy cập trái phép) |
| `no ip domain-lookup` | Tắt tính năng tự động phân giải DNS khi gõ nhầm lệnh — nếu không tắt, gõ sai lệnh sẽ khiến CLI treo vài chục giây chờ DNS timeout |
| `logging synchronous` | Đồng bộ log hệ thống với dòng lệnh đang gõ, tránh log system chen ngang làm rối màn hình đang nhập |
| `exec-timeout 5 0` | Tự động ngắt phiên sau 5 phút không hoạt động — biện pháp bảo mật cơ bản chống bỏ quên phiên đăng nhập |
| `line vty 0 15` | Cấu hình cho 16 phiên kết nối từ xa đồng thời (Telnet/SSH) — số lượng cụ thể tùy dòng thiết bị |

## 8. Cấu hình SSH quản trị từ xa

```
SW-ACCESS-01(config)# ip domain-name congty.local
SW-ACCESS-01(config)# username admin privilege 15 secret Admin@2026
SW-ACCESS-01(config)# crypto key generate rsa
% Chọn độ dài key >= 1024 bit khi được hỏi (khuyến nghị 2048 bit)
SW-ACCESS-01(config)# ip ssh version 2
SW-ACCESS-01(config)# line vty 0 15
SW-ACCESS-01(config-line)# transport input ssh
SW-ACCESS-01(config-line)# login local
SW-ACCESS-01(config-line)# exit
```

**Thứ tự bắt buộc**: phải đặt `hostname` và `ip domain-name` **trước** khi chạy `crypto key generate rsa` — vì RSA key được sinh dựa trên tên đầy đủ của thiết bị (hostname + domain-name), thiếu 1 trong 2 lệnh này sẽ không tạo được key và SSH sẽ không hoạt động. Lệnh `transport input ssh` giới hạn VTY chỉ nhận kết nối SSH, chặn hoàn toàn Telnet — kết hợp `login local` để xác thực bằng tài khoản cục bộ (username/secret) thay vì chỉ dùng password đơn giản như mục 7.

## 9. Cấu hình Interface

```
SW-ACCESS-01(config)# interface gigabitEthernet 0/1
SW-ACCESS-01(config-if)# description Ket noi PC ke toan 01
SW-ACCESS-01(config-if)# speed 1000
SW-ACCESS-01(config-if)# duplex full
SW-ACCESS-01(config-if)# no shutdown
SW-ACCESS-01(config-if)# exit

SW-ACCESS-01(config)# interface range gigabitEthernet 0/2 - 24
SW-ACCESS-01(config-if-range)# switchport mode access
SW-ACCESS-01(config-if-range)# no shutdown
```

`interface range` cho phép áp cùng lúc một cấu hình cho nhiều cổng liên tiếp — tiết kiệm đáng kể thời gian so với cấu hình từng cổng một khi triển khai hàng chục/hàng trăm cổng giống nhau. Lệnh `speed`/`duplex` để cứng (hard-code) chỉ nên dùng khi có lý do cụ thể (ví dụ thiết bị đầu kia không hỗ trợ auto-negotiation tốt) — mặc định nên để `auto` để tránh lỗi **duplex mismatch** đã học ở mục 4.4, [[Bai-03_Phuong-tien-truyen-dan-mang|Bài 3]].

## 10. CDP — Cisco Discovery Protocol

CDP là giao thức độc quyền của Cisco giúp các thiết bị Cisco tự động nhận diện lẫn nhau trên cùng đoạn mạng (model, IP, cổng kết nối...) mà không cần cấu hình gì thêm — rất hữu ích khi vẽ sơ đồ mạng thực tế hoặc xác định thiết bị nào cắm vào cổng nào.

```
SW-ACCESS-01# show cdp neighbors
SW-ACCESS-01# show cdp neighbors detail
```

> [!question] CDP hữu ích nhưng có nên bật mặc định trên mọi cổng không?
> Không nên bật trên các cổng hướng ra ngoài (uplink tới ISP, cổng kết nối với mạng không tin cậy) — vì CDP quảng bá thông tin thiết bị (model, phiên bản IOS, hostname) mà kẻ tấn công có thể thu thập để dò lỗ hổng đúng phiên bản. Dùng `no cdp run` (tắt toàn switch) hoặc `no cdp enable` (tắt từng cổng) cho các cổng nhạy cảm — đây là một phần của nguyên tắc hardening thiết bị sẽ học sâu hơn ở Bài 27.

## 11. Sao lưu và phục hồi cấu hình

| Lệnh | Công dụng |
|---|---|
| `copy running-config startup-config` (hoặc `write memory`/`wr`) | Lưu cấu hình đang chạy (RAM) xuống NVRAM — **bắt buộc** sau mỗi lần thay đổi, nếu không mất điện sẽ mất toàn bộ thay đổi |
| `copy startup-config running-config` | Nạp lại cấu hình đã lưu vào cấu hình đang chạy (ít dùng, thường dùng `reload` thay thế) |
| `copy running-config tftp` | Sao lưu cấu hình ra một TFTP server bên ngoài — thực hành chuẩn để có bản backup off-device |
| `show running-config` | Xem cấu hình đang chạy hiện tại (RAM) |
| `show startup-config` | Xem cấu hình đã lưu (NVRAM) — dùng để đối chiếu xem có thay đổi nào chưa được lưu |
| `erase startup-config` + `reload` | Xóa cấu hình khởi động. Lưu ý: trên Catalyst lệnh này **không** xóa VLAN database (`flash:vlan.dat`) — muốn về factory default thật sự phải chạy thêm `delete flash:vlan.dat` trước khi `reload`, nếu không VLAN/VTP cũ vẫn còn |

## 12. MAC Address Table

```
SW-ACCESS-01# show mac address-table
SW-ACCESS-01# show mac address-table dynamic
SW-ACCESS-01# clear mac address-table dynamic
```

Lệnh `show mac address-table` hiển thị đúng cơ chế học MAC đã học ở mục 4.1, [[Bai-04_Thiet-bi-ket-noi-mang|Bài 4]] — mỗi dòng thể hiện {VLAN : MAC : cổng}. Đây là lệnh đầu tiên cần chạy khi troubleshoot "thiết bị A không liên lạc được với thiết bị B trong cùng VLAN": xác nhận switch có học đúng MAC ở đúng cổng hay không trước khi nghi ngờ các nguyên nhân phức tạp hơn.

## 13. Verify & Troubleshoot

| Lệnh kiểm tra | Dùng để |
|---|---|
| `show version` | Xem phiên bản IOS, thời gian uptime, lý do lần reload gần nhất |
| `show running-config` | Đối chiếu toàn bộ cấu hình hiện tại |
| `show interfaces status` | Xem nhanh trạng thái tất cả cổng (up/down, VLAN, speed/duplex) trên 1 màn hình |
| `show interfaces gigabitEthernet 0/1` | Xem chi tiết 1 cổng: input/output error, collision, CRC error |
| `show mac address-table` | Kiểm tra bảng học MAC |
| `show cdp neighbors` | Xác định thiết bị lân cận |

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Không SSH vào được switch dù đã cấu hình | Thiếu `crypto key generate rsa`, hoặc thiếu `login local`, hoặc chưa đặt `ip domain-name` trước khi tạo key | Kiểm tra lại đúng thứ tự 4 bước ở mục 8 |
| Cổng lên nhưng tốc độ rất chậm | Duplex mismatch (mục 4.4, Bài 3) | `show interfaces` kiểm tra speed/duplex 2 đầu, đặt lại `auto` nếu không có lý do đặc biệt để hard-code |
| Sau khi mất điện, cấu hình vừa sửa biến mất | Quên `copy running-config startup-config` sau khi cấu hình | Luôn lưu ngay sau khi xác nhận cấu hình hoạt động đúng, không đợi "làm xong hết mới lưu" |
| CLI treo vài chục giây khi gõ nhầm lệnh | Thiếu `no ip domain-lookup`, switch cố phân giải DNS cho từ gõ sai | Thêm `no ip domain-lookup` vào cấu hình cơ bản ban đầu |

## 14. Lab thực hành

**Môi trường**: Cisco Packet Tracer.

1. Dựng 1 switch, đặt hostname, cấu hình `enable secret`, `service password-encryption`, banner motd, và bảo mật console/vty theo đúng mục 7.
2. Cấu hình SSH đầy đủ theo mục 8 (domain-name, username, crypto key, transport input ssh). Từ 1 PC, thử kết nối bằng SSH client (hoặc terminal trong Packet Tracer) — xác nhận đăng nhập được bằng tài khoản local.
3. Thử Telnet vào cùng switch — xác nhận bị từ chối kết nối vì đã giới hạn `transport input ssh`.
4. Cấu hình `interface range` cho một dải cổng, đặt `switchport mode access`, kiểm tra bằng `show interfaces status`.
5. Sau khi hoàn tất, `copy running-config startup-config`, sau đó `reload` thiết bị — xác nhận cấu hình vẫn còn nguyên sau khi khởi động lại (đối chiếu với trường hợp KHÔNG lưu để thấy rõ hậu quả).

**Kết quả kỳ vọng**: thành thạo toàn bộ chu trình từ cấu hình cơ bản đến bảo mật truy cập từ xa, và tự kiểm chứng được bằng thực hành tại sao bước `copy running-config startup-config` là bắt buộc chứ không phải tùy chọn.

## 15. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Cấu hình xong là tự động được lưu | Nhầm lẫn giữa running-config (RAM) và startup-config (NVRAM) | Mọi thay đổi chỉ nằm trong RAM cho đến khi chạy `copy running-config startup-config`, mất điện là mất hết |
| `enable password` và `enable secret` như nhau, dùng cái nào cũng được | Cả hai đều "đặt mật khẩu Privileged EXEC" | `enable password` lưu cleartext (không an toàn), `enable secret` tự động mã hóa — luôn ưu tiên dùng `enable secret`, nếu cả 2 cùng tồn tại thì `enable secret` được ưu tiên |
| Telnet vẫn đủ an toàn nếu chỉ dùng trong mạng nội bộ | Đánh giá thấp rủi ro từ chính nội bộ (nhân viên, thiết bị nhiễm mã độc, kẻ tấn công đã xâm nhập được 1 điểm) | Telnet luôn truyền cleartext bất kể mạng nội bộ hay không — chuẩn thực tế là dùng SSH ở mọi nơi |
| `service password-encryption` làm mật khẩu an toàn tuyệt đối | Đây chỉ là mã hóa yếu (type 7), có công cụ giải mã ngược công khai trong vài giây | Chỉ nên xem là lớp che chắn cơ bản (tránh nhìn thấy bằng mắt thường), không thay thế được `enable secret` cho các mật khẩu quan trọng (mặc định `enable secret` dùng type 5/MD5, hiện bị coi là yếu; Cisco khuyến nghị type 8/9, ví dụ `enable algorithm-type scrypt secret ...`) |

## 16. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Trình bày quy trình khởi động của switch Cisco, gắn với từng loại bộ nhớ.**
→ POST chạy từ ROM kiểm tra phần cứng → bootstrap (ROM) nạp IOS image từ Flash vào RAM → nạp startup-config từ NVRAM vào RAM (trở thành running-config) → switch sẵn sàng hoạt động.

**2. Vì sao phải `copy running-config startup-config` sau khi cấu hình, không tự động lưu?**
→ Mọi thay đổi CLI chỉ áp dụng trực tiếp vào running-config nằm trong RAM (volatile). Thiết kế tách biệt RAM/NVRAM giúp admin có thể thử nghiệm cấu hình, kiểm tra hoạt động đúng trước khi quyết định lưu vĩnh viễn — nếu tự động lưu, một lệnh sai sẽ ngay lập tức trở thành cấu hình khởi động, khó rollback hơn.

**3. Vì sao phải đặt hostname và ip domain-name trước khi chạy crypto key generate rsa?**
→ RSA key pair dùng cho SSH được sinh dựa trên tên đầy đủ của thiết bị (hostname.domain-name). Thiếu 1 trong 2 thông tin này, lệnh sẽ không tạo được key và SSH server không khởi động được.

**4. Phân biệt `enable password` và `enable secret`. Nếu cấu hình cả hai thì lệnh nào có hiệu lực?**
→ `enable password` lưu mật khẩu dạng cleartext (không an toàn). `enable secret` tự động mã hóa bằng hash mạnh hơn. Nếu cấu hình cả hai, `enable secret` luôn được ưu tiên áp dụng.

**5. `interface range` dùng để làm gì? Cho ví dụ tình huống thực tế cần dùng.**
→ Cho phép áp cùng một cấu hình cho nhiều cổng liên tiếp trong 1 lần thay vì lặp lại từng cổng — ví dụ khi cấu hình 24 cổng access cho toàn bộ dãy PC nhân viên cùng thuộc 1 VLAN, dùng `interface range gigabitEthernet 0/1 - 24` thay vì gõ 24 lần.

## 17. Tài liệu tham khảo

- [Compare Cisco Switches — Cisco](https://www.cisco.com/c/en/us/products/switches/switch-comparison.html)
- [Configure SSH on Routers — Cisco](https://www.cisco.com/c/en/us/support/docs/security-vpn/secure-shell-ssh/4145-ssh.html)
- [Heimdal Security — A Technical Analysis of the Mirai Botnet Phenomenon](https://heimdalsecurity.com/blog/mirai-botnet-phenomenon/)
- [ComputingForGeeks — Configure Cisco Device Access Control](https://computingforgeeks.com/cisco-device-access-control-passwords/)

---
**Liên kết**: [[Bai-06_Dia-chi-IPv6|← Bài 6]] · [[Bai-08_Cau-hinh-va-quan-ly-VLAN|Bài 8 - Cấu hình và quản lý VLAN →]]
