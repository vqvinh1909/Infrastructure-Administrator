# Mini Lab — Module 01: Nền tảng mạng (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-01.md`

---

## 1. Chọn phương tiện truyền dẫn

| Đoạn | Khoảng cách | Loại cáp/media đề xuất | Lý do |
|---|---|---|---|
| (A) Server Room → Tầng 2 | 55m | UTP Cat6 (đồng) | Trong giới hạn 100m của Ethernet đồng, không có nguồn nhiễu điện gần đó — không cần chi phí cáp quang. Cat6 chọn thay vì Cat5e để có headroom nâng cấp 10GbE sau này. |
| (B) Server Room → Tầng 1 | 95m | Cáp quang Multimode | 95m vượt giới hạn cáp đồng theo TIA-568 (permanent link tối đa 90m; channel tối đa 100m đã gồm cả patch cord hai đầu) nên UTP không đạt chuẩn; ngoài ra đoạn này đi gần động cơ thang máy — nguồn nhiễu điện từ (EMI) mạnh làm suy giảm tín hiệu đồng. Cáp quang miễn nhiễm hoàn toàn với EMI, đảm bảo ổn định. |
| (C) Tòa chính → Tòa kho | 300m | Cáp quang Single-mode | Vượt xa giới hạn 100m của cáp đồng nên phải dùng quang. Multimode OM3 cũng đủ cho 300m (10GBASE-SR hỗ trợ tới 300m), nhưng chọn Single-mode vì dễ mở rộng xa hơn sau này; quan trọng hơn — cáp quang không dẫn điện nên loại bỏ hoàn toàn nguy cơ sét lan truyền giữa 2 tòa nhà không có đường điện chung, khác với cáp đồng có thể dẫn xung sét gây cháy thiết bị. |

## 2. Bảng VLSM — `192.168.10.0/24`

Thứ tự cấp phát: sắp theo nhu cầu giảm dần (25 → 12 → 10 → 8 → 2) để tránh vỡ khối.

| Subnet | Host cần | Network | Mask (/CIDR) | Host dùng được | Broadcast |
|---|---|---|---|---|---|
| Sales + Lễ tân | 25 | 192.168.10.0 | 255.255.255.224 (/27) | .1 – .30 (30 host) | .31 |
| Kế toán + Nhân sự | 12 | 192.168.10.32 | 255.255.255.240 (/28) | .33 – .46 (14 host) | .47 |
| Guest Wifi | 10 | 192.168.10.48 | 255.255.255.240 (/28) | .49 – .62 (14 host) | .63 |
| IT + Server | 8 | 192.168.10.64 | 255.255.255.240 (/28) | .65 – .78 (14 host) | .79 |
| Link WAN (point-to-point) | 2 | 192.168.10.80 | 255.255.255.252 (/30) | .81 – .82 (2 host) | .83 |
| Dự phòng mở rộng | — | 192.168.10.84 – .255 | — (còn 172 địa chỉ, đủ cấp thêm nhiều /27–/25) | — | — |

Kiểm tra không chồng lấn: các khối `.0/27`, `.32/28`, `.48/28`, `.64/28`, `.80/30` nằm liên tiếp, không giao nhau, đúng ranh giới nhị phân của từng prefix.

## 3. Bảng IPv6 — chia từ `2001:db8:aabb::/48`

| Tầng/khu vực | Subnet IPv6 (/64) |
|---|---|
| Tầng 1 (Sales/Guest) | `2001:db8:aabb:1::/64` |
| Tầng 2 (Kế toán/NS) | `2001:db8:aabb:2::/64` |
| Tầng 3 (IT/Server) | `2001:db8:aabb:3::/64` |

Mỗi subnet dùng đúng 1 trong 65536 khối `/64` khả dụng của `/48` (nhóm 16-bit (hextet) thứ 4 — chính là subnet ID — tăng dần: `:1:`, `:2:`, `:3:`) — không đụng vào phần Interface ID (64 bit cuối) dành cho host.

## 4. Giải thích các điểm quan trọng / dễ sai

- **Sai lầm phổ biến (A/B):** nhiều người nhầm giới hạn 100m là chiều dài cáp ngang (thực tế permanent link chỉ 90m) và bỏ qua yếu tố nhiễu điện — 95m đã vượt giới hạn 90m của cáp đồng theo TIA-568, lại còn gần động cơ thang máy nên phải dùng quang.
- **Sai lầm phổ biến (VLSM):** cấp `/27` (30 host) cho mọi subnet vì "cho chắc" — lãng phí gần 20 địa chỉ ở nhóm 8-10 người, và dễ hết khối `/24` khi công ty mở rộng thêm phòng ban.
- **Sai lầm phổ biến (WAN link):** dùng `/24` hoặc `/29` cho link point-to-point 2 host — `/30` là tối ưu (chỉ 2 host dùng được, không lãng phí).
- **IPv6:** khác IPv4, không cần "tiết kiệm" địa chỉ trong từng `/64` — điểm cần đúng là ranh giới subnet nằm ở đâu trong 16 bit subnet ID của `/48`, không phải số lượng host.
