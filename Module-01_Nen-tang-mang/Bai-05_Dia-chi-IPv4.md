---
title: Bài 5 - Địa chỉ IP Version 4
type: Foundation
difficulty: Beginner to Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — subnetting tay là bài test thực hành bắt buộc trong hầu hết buổi phỏng vấn Networking"
tags: [networking, fundamentals, ipv4, subnetting, vlsm, flsm, cidr, sysadmin]
aliases: [IPv4, Subnetting, VLSM, FLSM, Wildcard Mask]
related: ["[[Bai-04_Thiet-bi-ket-noi-mang]]", "[[Bai-06_Dia-chi-IPv6]]"]
last_review: 2026-07-21
---

# Bài 5: Địa chỉ IP Version 4

> [!important] Tại sao cần file này
> Subnetting không phải bài toán học thuật — đây là kỹ năng dùng hàng ngày khi quy hoạch mạng cho một tòa nhà mới, tách VLAN theo phòng ban, hay đơn giản là đọc hiểu vì sao 2 thiết bị "cùng dải IP" mà không ping được nhau. Gần như 100% buổi phỏng vấn Networking/Sysadmin có ít nhất 1 câu subnetting tính tay tại chỗ — không phải để làm khó, mà vì đây là phép thử nhanh nhất cho việc ứng viên có thực sự hiểu cấu trúc địa chỉ IP hay chỉ học thuộc lòng.

## 1. Cấu trúc và phân lớp IPv4

Địa chỉ IPv4 dài 32-bit, biểu diễn thành 4 nhóm 8-bit (octet) cách nhau bằng dấu chấm, mỗi octet có giá trị 0-255 (dotted decimal) — ví dụ `192.168.1.10`. Về mặt logic, một địa chỉ IP luôn chia thành 2 phần: **Network portion** (phần mạng — xác định địa chỉ đó thuộc mạng nào) và **Host portion** (phần host — xác định thiết bị cụ thể trong mạng đó).

| Lớp | Dải địa chỉ | Subnet mask mặc định | Số host tối đa | Mục đích |
|---|---|---|---|---|
| A | 1.0.0.0 – 126.255.255.255 | /8 (255.0.0.0) | ~16.7 triệu | Mạng cực lớn (ISP lớn, tổ chức quốc tế) |
| B | 128.0.0.0 – 191.255.255.255 | /16 (255.255.0.0) | ~65.500 | Mạng vừa (trường đại học, doanh nghiệp lớn) |
| C | 192.0.0.0 – 223.255.255.255 | /24 (255.255.255.0) | 254 | Mạng nhỏ (văn phòng, LAN gia đình) |
| D | 224.0.0.0 – 239.255.255.255 | — | — | Multicast (mục 3) |
| E | 240.0.0.0 – 255.255.255.255 | — | — | Dự trữ/thử nghiệm, không dùng thực tế |

Riêng dải `127.0.0.0/8` được dành riêng cho loopback (địa chỉ tự trỏ về chính thiết bị, thường dùng `127.0.0.1`) — không thuộc phân lớp A dùng được cho host thật.

## 2. IP Private vs IP Public

**IP Public** là địa chỉ định tuyến được trên Internet, duy nhất toàn cầu, do các tổ chức quản lý số hiệu Internet (IANA/RIR) cấp phát. **IP Private** chỉ có ý nghĩa trong phạm vi mạng nội bộ, không định tuyến được ra Internet, và theo RFC 1918 gồm 3 dải:

| Dải Private (RFC 1918) | Subnet mặc định | Số địa chỉ | Thường dùng cho |
|---|---|---|---|
| 10.0.0.0 – 10.255.255.255 | /8 | ~16.7 triệu | Doanh nghiệp/cloud lớn, cần không gian địa chỉ rộng |
| 172.16.0.0 – 172.31.255.255 | /12 | ~1 triệu | Doanh nghiệp vừa, dải VPN |
| 192.168.0.0 – 192.168.255.255 | /16 | 65.536 | Văn phòng nhỏ, thiết bị gia dụng |

Vì các dải này được dùng lặp lại ở hàng triệu mạng nội bộ khác nhau trên thế giới mà không xung đột, thiết bị dùng IP Private muốn ra Internet phải thông qua kỹ thuật NAT (Network Address Translation — Bài 20) để đổi sang IP Public tại điểm ra ngoài.

## 3. Unicast, Multicast, Broadcast

- **Unicast**: gửi từ 1 nguồn đến đúng 1 đích cụ thể — kiểu giao tiếp phổ biến nhất (duyệt web, gửi email).
- **Multicast**: gửi từ 1 nguồn đến một nhóm thiết bị đã đăng ký nhận (không phải tất cả) — dùng dải `224.0.0.0 – 239.255.255.255` (Class D), ứng dụng điển hình: streaming video hội nghị, giao thức định tuyến trao đổi thông tin giữa các router (OSPF dùng địa chỉ multicast `224.0.0.5`/`224.0.0.6`).
- **Broadcast**: gửi từ 1 nguồn đến **toàn bộ** thiết bị trong cùng broadcast domain (địa chỉ toàn bit 1 trong phần host, ví dụ `192.168.1.255` cho mạng `192.168.1.0/24`) — ví dụ điển hình là ARP Request đã học ở [[Bai-02_Mo-hinh-OSI-TCPIP|Bài 2]].

## 4. Ý nghĩa các thông số cấu hình IP trên thiết bị

Khi cấu hình IP tĩnh cho một máy tính, luôn cần đủ 4 thông số sau — thiếu một trong số này đều gây ra lỗi kết nối với triệu chứng khác nhau:

| Thông số | Vai trò | Nếu sai/thiếu thì sao |
|---|---|---|
| IP Address | Định danh thiết bị trong mạng | Trùng IP với thiết bị khác → xung đột địa chỉ (IP conflict), cả 2 máy mất kết nối chập chờn |
| Subnet Mask | Xác định ranh giới Network/Host, máy tự biết đích cùng mạng hay khác mạng | Sai mask → máy tính sai phạm vi mạng, gửi nhầm traffic cùng mạng ra gateway hoặc ngược lại |
| Default Gateway | Địa chỉ router mà máy gửi traffic đến khi đích nằm ở mạng khác | Không có/sai gateway → chỉ liên lạc được trong cùng mạng, không ra được mạng khác/Internet |
| DNS Server | Địa chỉ máy chủ phân giải tên miền sang IP | Sai DNS → không mở được website bằng tên miền dù vẫn ping được bằng IP trực tiếp |

## 5. CIDR và Subnet Mask

CIDR (Classless Inter-Domain Routing) biểu diễn subnet mask bằng số bit mạng sau dấu `/` (prefix length) thay vì viết đầy đủ 4 octet — ví dụ `/24` tương đương `255.255.255.0`.

| Prefix (CIDR) | Subnet Mask | Số bit host | Số host khả dụng (trừ Network + Broadcast) |
|---|---|---|---|
| /24 | 255.255.255.0 | 8 | 254 |
| /25 | 255.255.255.128 | 7 | 126 |
| /26 | 255.255.255.192 | 6 | 62 |
| /27 | 255.255.255.224 | 5 | 30 |
| /28 | 255.255.255.240 | 4 | 14 |
| /29 | 255.255.255.248 | 3 | 6 |
| /30 | 255.255.255.252 | 2 | 2 (dùng cho liên kết điểm-điểm giữa 2 router) |

Công thức cốt lõi cần thuộc lòng: **số host khả dụng = 2^(số bit host) − 2** (trừ đi 2 vì địa chỉ đầu dải luôn là Network Address, địa chỉ cuối dải luôn là Broadcast Address — cả hai đều không gán được cho thiết bị).

## 6. Chia mạng con — FLSM (Fixed Length Subnet Mask)

FLSM chia một mạng lớn thành nhiều mạng con **có kích thước bằng nhau**, bất kể mỗi mạng con thực tế cần bao nhiêu host.

**Ví dụ**: chia `192.168.1.0/24` thành 4 mạng con bằng nhau → mượn 2 bit từ phần host (2² = 4 mạng con) → mỗi mạng con thành `/26` (62 host khả dụng):

```
192.168.1.0/26    →  192.168.1.0   – 192.168.1.63   (host: .1 – .62,  broadcast: .63)
192.168.1.64/26   →  192.168.1.64  – 192.168.1.127  (host: .65 – .126, broadcast: .127)
192.168.1.128/26  →  192.168.1.128 – 192.168.1.191  (host: .129 – .190, broadcast: .191)
192.168.1.192/26  →  192.168.1.192 – 192.168.1.255  (host: .193 – .254, broadcast: .255)
```

**Nhược điểm cốt lõi**: nếu một phòng ban chỉ có 5 máy nhưng vẫn nhận nguyên 1 subnet `/26` (62 host), 57 địa chỉ còn lại bị lãng phí hoàn toàn không dùng được cho phòng ban khác — đây chính là lý do VLSM ra đời.

## 7. Chia mạng con — VLSM (Variable Length Subnet Mask)

VLSM cho phép chia các mạng con **kích thước khác nhau tùy theo nhu cầu thực tế** của từng khu vực, tránh lãng phí địa chỉ. Nguyên tắc: **luôn cấp phát từ nhu cầu lớn nhất đến nhỏ nhất** để tránh xung đột dải địa chỉ.

**Ví dụ thực chiến**: quy hoạch `192.168.1.0/24` cho 1 tòa nhà văn phòng có: phòng Kinh doanh (50 máy), phòng IT (20 máy), phòng HR (10 máy), và 1 liên kết điểm-điểm giữa 2 router (2 địa chỉ).

| Bước | Nhu cầu | Số bit host cần | Prefix | Dải cấp phát | Host khả dụng |
|---|---|---|---|---|---|
| 1 | Kinh doanh — 50 máy | 2⁶−2=62 ≥ 50 → 6 bit | /26 | 192.168.1.0 – .63 | .1 – .62 |
| 2 | IT — 20 máy | 2⁵−2=30 ≥ 20 → 5 bit | /27 | 192.168.1.64 – .95 | .65 – .94 |
| 3 | HR — 10 máy | 2⁴−2=14 ≥ 10 → 4 bit | /28 | 192.168.1.96 – .111 | .97 – .110 |
| 4 | Liên kết router-router — 2 địa chỉ | 2²−2=2 ≥ 2 → 2 bit | /30 | 192.168.1.112 – .115 | .113 – .114 |

```
192.168.1.0/24 (256 địa chỉ gốc)
  |
  |-- 192.168.1.0/26    (64) -> Kinh doanh, dùng 50/62
  |-- 192.168.1.64/27   (32) -> IT, dùng 20/30
  |-- 192.168.1.96/28   (16) -> HR, dùng 10/14
  |-- 192.168.1.112/30  (4)  -> Link router, dùng 2/2
  `-- 192.168.1.116 - .255   -> còn dư, dự trữ cho mở rộng
```

So với FLSM (mỗi phòng ban đều phải nhận nguyên 1 subnet `/26` dù nhu cầu khác nhau rất nhiều), VLSM tiết kiệm được phần lớn không gian địa chỉ còn dư để dự trữ cho việc mở rộng sau này — đây là kỹ thuật quy hoạch chuẩn dùng trong thực tế doanh nghiệp, không phải FLSM.

> [!example] Vấn đề thực chiến — Xung đột dải IP khi sáp nhập doanh nghiệp (M&A)
> Một nghiên cứu về quy hoạch IP doanh nghiệp ghi nhận: khi 2 công ty sáp nhập, việc cả 2 bên cùng dùng chung dải Private phổ biến (ví dụ cả 2 đều quy hoạch quanh `10.1.0.0/16`) dẫn đến **xung đột dải địa chỉ (overlapping subnet)** — router không thể định tuyến chính xác vì cùng một dải IP tồn tại ở 2 vị trí vật lý khác nhau, một số thiết bị ở 2 nơi vô tình trùng cả IP. Giải pháp thường phải dùng NAT tạm thời giữa 2 mạng hoặc quy hoạch lại (renumber) toàn bộ một bên — cả hai đều tốn nhiều thời gian/rủi ro downtime. Bài học quy hoạch: khi thiết kế IP cho doanh nghiệp, nên tránh các dải "quá phổ biến" như `192.168.1.0/24` mặc định của thiết bị gia dụng, và cân nhắc khả năng sáp nhập/kết nối với mạng khác trong tương lai ngay từ giai đoạn quy hoạch ban đầu. Nguồn: [Temple University — IP Address Consolidation and Reconfiguration in Enterprise Networks](https://cis.temple.edu/~apal/ic3n_ip_consolidation_2016.pdf).

## 8. Wildcard Mask

Wildcard mask là "nghịch đảo" của subnet mask (đảo bit 0 thành 1 và ngược lại), dùng trong cấu hình Access Control List (ACL — Bài 16-17) và một số giao thức định tuyến (OSPF — Bài 14) thay vì dùng subnet mask trực tiếp.

```
Subnet mask /24:    255.255.255.0    = 11111111.11111111.11111111.00000000
Wildcard tương ứng:  0.0.0.255        = 00000000.00000000.00000000.11111111
```

Ý nghĩa: bit **0** trong wildcard mask nghĩa là "phải khớp chính xác", bit **1** nghĩa là "bỏ qua, không cần khớp" — ngược hoàn toàn với logic của subnet mask (bit 1 = phần mạng phải khớp, bit 0 = phần host được tự do). Đây là điểm gây nhầm lẫn phổ biến nhất với người mới học ACL.

## 9. Quy hoạch IPv4 cho tổ chức, doanh nghiệp

Quy trình quy hoạch thực tế gồm các bước:

1. **Thu thập yêu cầu**: liệt kê từng khu vực/phòng ban/VLAN cần bao nhiêu thiết bị hiện tại.
2. **Dự trù tăng trưởng**: luôn cộng thêm hệ số dự phòng (thường 30-50%) cho từng khu vực — vì đổi prefix của một subnet đang chạy (renumbering) tốn kém và rủi ro downtime hơn nhiều so với quy hoạch dư ngay từ đầu.
3. **Áp dụng VLSM** theo thứ tự từ nhu cầu lớn nhất đến nhỏ nhất (mục 7) để tối ưu không gian địa chỉ.
4. **Dành riêng dải cho hạ tầng quản lý** (management network — đã nhắc ở mục 5.5, [[Bai-01_Gioi-thieu-mang-may-tinh|Bài 1]]) tách biệt khỏi dải người dùng.
5. **Ghi chép lại (IP Address Management — IPAM)**: dù dùng file Excel đơn giản hay công cụ IPAM chuyên dụng, việc ghi lại ai/thiết bị nào dùng dải nào là bắt buộc để tránh cấp phát trùng lặp về sau.

## 10. Các lệnh kiểm tra IP và kết nối

| Lệnh | Hệ điều hành | Công dụng |
|---|---|---|
| `ping <IP/domain>` | Windows/Linux | Kiểm tra kết nối tới đích bằng ICMP Echo Request/Reply |
| `ipconfig` (Windows) / `ifconfig` hoặc `ip addr` (Linux) | — | Xem cấu hình IP, subnet mask, gateway hiện tại của máy |
| `tracert <IP/domain>` (Windows) / `traceroute` (Linux) | — | Hiển thị từng chặng router (hop) mà gói tin đi qua trước khi tới đích, giúp xác định đoạn nào trên đường đi bị nghẽn/mất gói |

## 11. Lab thực hành

**Môi trường**: Cisco Packet Tracer + máy tính cá nhân (Windows/Linux).

1. Trên máy thật, chạy `ipconfig`/`ip addr` ghi lại IP, subnet mask, gateway, DNS hiện tại.
2. `ping` một địa chỉ ngoài Internet (ví dụ `8.8.8.8`), sau đó `tracert`/`traceroute` cùng địa chỉ đó — quan sát số hop và địa chỉ IP của từng router trung gian.
3. Trong Packet Tracer, dựng lại đúng mô hình VLSM ở mục 7 (1 router, 4 nhánh: Kinh doanh/IT/HR/Link router-router), gán IP theo đúng bảng đã tính, xác nhận ping thông trong từng nhánh và giữa các nhánh qua router.
4. Cố tình gán sai subnet mask cho 1 PC (ví dụ dùng `/24` thay vì `/26` đúng), quan sát hiện tượng mất kết nối dù IP vẫn đúng dải — minh chứng trực quan cho vai trò của subnet mask ở mục 4.

**Kết quả kỳ vọng**: tính đúng tay được bảng VLSM, cấu hình thành công trên Packet Tracer, và tự giải thích được bằng lời vì sao sai subnet mask gây mất kết nối dù địa chỉ IP không đổi.

## 12. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Cứ "IP cùng dải" là chắc chắn ping thông | Bỏ qua vai trò của subnet mask trong việc xác định ranh giới mạng | 2 địa chỉ có thể "trông giống nhau" nhưng nằm ở 2 subnet khác nhau nếu mask khác nhau — phải tính toán Network Address để xác nhận thực sự cùng mạng |
| FLSM và VLSM là 2 tên gọi khác nhau của cùng 1 kỹ thuật | Nhầm lẫn vì cả 2 đều "chia mạng con" | FLSM chia đều kích thước (lãng phí nếu nhu cầu không đều), VLSM chia theo đúng nhu cầu thực tế từng khu vực |
| Wildcard mask hoạt động giống hệt subnet mask, chỉ khác cách viết | Nhầm lẫn ý nghĩa bit 0/1 | Logic đảo ngược hoàn toàn: subnet mask bit 1 = phải khớp, wildcard mask bit 1 = bỏ qua không cần khớp |
| Chỉ cần đúng IP và subnet mask là đủ, gateway/DNS không quan trọng bằng | Đánh giá thấp vai trò 2 thông số còn lại | Thiếu gateway → không ra khỏi mạng cục bộ được; thiếu/sai DNS → không dùng được tên miền dù IP vẫn hoạt động bình thường |

## 13. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Cho địa chỉ 192.168.1.130/26. Xác định Network Address, Broadcast Address, và dải host khả dụng.**
→ /26 = 255.255.255.192, bước nhảy (block size) = 256−192 = 64. Các dải: .0-.63, .64-.127, .128-.191... → 192.168.1.130 rơi vào dải .128-.191. Network Address: 192.168.1.128. Broadcast Address: 192.168.1.191. Host khả dụng: 192.168.1.129 – 192.168.1.190.

**2. Vì sao VLSM tiết kiệm địa chỉ hơn FLSM? Cho ví dụ.**
→ FLSM chia đều kích thước bất kể nhu cầu thực tế — ví dụ chia 4 subnet /26 bằng nhau dù có nhánh chỉ cần 2 host (link router-router) vẫn nhận nguyên 62 host, lãng phí 60 địa chỉ. VLSM cấp đúng theo nhu cầu (link router-router chỉ cần /30 = 2 host), phần dư dùng cho mục đích khác hoặc dự trữ.

**3. Phân biệt IP Private và IP Public. Vì sao thiết bị dùng IP Private vẫn ra Internet được?**
→ IP Public định tuyến được trên Internet, duy nhất toàn cầu. IP Private (RFC 1918) chỉ có ý nghĩa nội bộ, dùng lặp lại được ở nhiều mạng khác nhau. Thiết bị IP Private ra Internet được nhờ NAT đổi sang IP Public tại thiết bị biên (router/firewall).

**4. Wildcard mask 0.0.0.15 tương ứng với subnet mask nào? Bao nhiêu địa chỉ bị ảnh hưởng?**
→ Đảo bit: 0.0.0.15 = 00000000.00000000.00000000.00001111 → subnet mask tương ứng = 255.255.255.240 (/28). Có 16 địa chỉ (4 bit host = 2⁴) nằm trong phạm vi mà wildcard này cho phép khớp.

**5. Máy A và máy B cùng đặt IP 192.168.1.10 nhưng máy A dùng mask /24, máy B dùng mask /25. Có vấn đề gì không?**
→ Có — đây là ví dụ về cấu hình sai gây hiểu nhầm phạm vi mạng: dù cùng IP và có vẻ "cùng dải", 2 máy đang tự nhận thức phạm vi broadcast domain khác nhau (một bên nghĩ mạng có 254 host, bên kia nghĩ chỉ có 126 host), có thể dẫn đến việc một bên gửi thẳng ra gateway những gói tin lẽ ra nên gửi trực tiếp trong mạng cục bộ, hoặc ngược lại.

## 14. Tài liệu tham khảo

- [RFC 1918 — Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918)
- [IPv4 Private Address Ranges: RFC 1918 Explained](https://www.ippigly.com/blog/ipv4-private-ranges)
- [Temple University — IP Address Consolidation and Reconfiguration in Enterprise Networks](https://cis.temple.edu/~apal/ic3n_ip_consolidation_2016.pdf)

---
**Liên kết**: [[Bai-04_Thiet-bi-ket-noi-mang|← Bài 4]] · [[Bai-06_Dia-chi-IPv6|Bài 6 - Địa chỉ IP Version 6 →]]
