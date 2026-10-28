---
title: Bài 6 - Địa chỉ IP Version 6
type: Foundation
difficulty: Beginner to Intermediate
lab_required: true
exam: "Sysadmin Junior/Mid-level — IPv6 không còn là 'công nghệ tương lai', đã là hạ tầng hiện tại"
tags: [networking, fundamentals, ipv6, unicast, multicast, anycast, sysadmin]
aliases: [IPv6, Global Unicast, Link-Local, ULA]
related: ["[[Bai-05_Dia-chi-IPv4]]", "[[Bai-01_Gioi-thieu-mang-may-tinh]]"]
last_review: 2026-07-21
---

# Bài 6: Địa chỉ IP Version 6

> [!important] Tại sao cần file này
> [[Bai-05_Dia-chi-IPv4|Bài 5]] đã nhắc đến việc kho địa chỉ IPv4 toàn cầu cạn kiệt từ 03/02/2011. IPv6 ra đời chính để giải quyết vấn đề đó, và đây không còn là câu chuyện lý thuyết: theo Google, lượng truy cập qua IPv6 thuần đã **vượt mốc 50%** lần đầu tiên vào ngày 28/03/2026 — nghĩa là hơn một nửa traffic Internet hiện nay đã chạy trên IPv6. Sysadmin ngày nay bắt buộc phải đọc hiểu và cấu hình được cả 2 giao thức song song (dual-stack), không thể chỉ biết IPv4.

## 1. Cấu trúc địa chỉ IPv6

Địa chỉ IPv6 dài **128-bit** (gấp 4 lần IPv4), biểu diễn thành 8 nhóm, mỗi nhóm 16-bit (gọi là **hextet**) viết dưới dạng số hex, cách nhau bằng dấu hai chấm:

```
2001:0db8:0000:0042:0000:8a2e:0370:7334
---- ---- ---- ---- ---- ---- ---- ----
  1    2    3    4    5    6    7    8
(mỗi nhóm = 1 hextet = 16 bit; 8 hextet x 16 bit = 128 bit)
```

Để hình dung độ lớn: IPv4 có khoảng 4,3 tỷ địa chỉ (2³²); IPv6 có khoảng 340 undecillion địa chỉ (2¹²⁸ ≈ 3,4×10³⁸) — đủ để cấp hàng nghìn tỷ tỷ địa chỉ cho mỗi người trên Trái Đất, giải quyết dứt điểm bài toán khan hiếm địa chỉ đã học ở Bài 5.

## 2. Quy tắc rút gọn (Compression)

Địa chỉ IPv6 đầy đủ rất dài và khó đọc, nên có 2 quy tắc rút gọn bắt buộc phải nắm:

**Quy tắc 1 — Bỏ số 0 đứng đầu mỗi nhóm:**
```
2001:0db8:0000:0042:0000:8a2e:0370:7334
→ 2001:db8:0:42:0:8a2e:370:7334
```

**Quy tắc 2 — Thay MỘT dãy liên tiếp các nhóm toàn số 0 bằng `::` (chỉ được dùng 1 lần trong toàn bộ địa chỉ):**
```
2001:db8:0:42:0:8a2e:370:7334
→ 2001:db8:0:42:0:8a2e:370:7334   (chỉ có 2 nhóm "0" đơn lẻ, không liên tiếp nên giữ nguyên)

Ví dụ khác — địa chỉ có nhiều nhóm 0 liên tiếp:
2001:0db8:0000:0000:0000:0000:0000:0001
→ 2001:db8::1
```

> [!question] Vì sao `::` chỉ được dùng đúng 1 lần trong 1 địa chỉ?
> Vì nếu dùng 2 lần, máy tính không thể xác định mỗi cụm `::` đại diện cho bao nhiêu nhóm số 0 — gây ra địa chỉ mơ hồ (ambiguous), không thể khôi phục lại chính xác dạng đầy đủ. Đây là lỗi cú pháp phổ biến nhất khi người mới viết tay địa chỉ IPv6.

## 3. Phân loại địa chỉ IPv6: Unicast, Multicast, Anycast

Về khái niệm, Unicast/Multicast tương tự IPv4 (mục 3, [[Bai-05_Dia-chi-IPv4|Bài 5]]) nhưng IPv6 bổ sung Anycast và **loại bỏ hoàn toàn Broadcast** — thay thế bằng Multicast phạm vi hẹp để tránh việc phát tràn lan không cần thiết.

### 3.1 Unicast — chia thành 3 loại con

| Loại | Dải/Prefix | Vai trò | Tương đương bên IPv4 |
|---|---|---|---|
| **Global Unicast** | `2000::/3` | Địa chỉ định tuyến được trên Internet, duy nhất toàn cầu | IP Public |
| **Link-Local** | `FE80::/10` | Chỉ có ý nghĩa trong cùng 1 đoạn mạng vật lý (không định tuyến qua router), tự động sinh ra trên **mọi** interface có IPv6 | Không có tương đương trực tiếp — luôn tồn tại song song với địa chỉ khác |
| **Unique Local (ULA)** | `FC00::/7` | Dùng nội bộ tổ chức, không định tuyến ra Internet công cộng | IP Private (RFC 1918) |

### 3.2 Multicast

Bắt đầu bằng `FF00::/8`, dùng để gửi đến một nhóm thiết bị đã đăng ký — thay thế hoàn toàn vai trò của Broadcast trong IPv4. Ví dụ quan trọng nhất: `FF02::1` là địa chỉ multicast đại diện cho **"tất cả node trên link"** (all-nodes), dùng trong cơ chế tự động cấu hình địa chỉ (mục 6).

### 3.3 Anycast

Nhiều thiết bị khác nhau cùng dùng chung 1 địa chỉ Anycast; gói tin gửi đến địa chỉ đó sẽ được định tuyến đến node **gần nhất** về mặt topology (không phải gần nhất về khoảng cách vật lý). Ứng dụng thực tế phổ biến nhất: các DNS root server và CDN lớn dùng Anycast để cùng một địa chỉ IP trả lời từ nhiều vị trí trên thế giới, giúp giảm độ trễ và tăng khả năng chịu lỗi (nếu 1 node hỏng, traffic tự động chuyển sang node gần nhất khác mà client không cần đổi địa chỉ đích).

> [!question] Vì sao IPv6 bỏ hẳn Broadcast?
> Broadcast trong IPv4 buộc mọi thiết bị trong mạng phải xử lý gói tin dù không liên quan (gây lãng phí tài nguyên CPU trên toàn bộ thiết bị) — đây chính là nguyên nhân gốc của hiện tượng broadcast storm đã học ở Bài 2. IPv6 thiết kế lại bằng Multicast phạm vi hẹp (chỉ đúng nhóm thiết bị cần nhận, ví dụ chỉ router hoặc chỉ tất cả node) — vừa giữ được chức năng tương đương vừa tránh lãng phí tài nguyên trên các thiết bị không liên quan.

## 4. Subnet IPv6

IPv6 dùng prefix length giống CIDR của IPv4 nhưng với quy ước khác về kích thước tiêu chuẩn:

| Prefix | Dùng cho | Ghi chú |
|---|---|---|
| `/48` | Cấp phát cho 1 tổ chức/site | Một tổ chức thường được ISP cấp nguyên khối `/48` để tự chia nhỏ |
| `/64` | 1 subnet/1 VLAN (LAN segment) | **Chuẩn mặc định** cho mọi subnet có host thông thường — gần như luôn dùng /64, không tùy chỉnh nhỏ hơn như IPv4 |
| `/127` | Liên kết điểm-điểm giữa 2 router | Tương đương vai trò /30 bên IPv4, chỉ cần 2 địa chỉ |

Khác biệt quan trọng nhất so với tư duy VLSM ở Bài 5: vì không gian địa chỉ IPv6 gần như vô hạn, **không cần tiết kiệm từng bit** như IPv4 — mọi subnet LAN thông thường đều dùng `/64` bất kể mạng đó có 5 hay 5.000 thiết bị, việc quy hoạch chuyển trọng tâm từ "tiết kiệm địa chỉ" sang "tổ chức phân cấp dễ quản lý" (ví dụ chia `/48` của công ty thành nhiều `/56` cho từng chi nhánh, mỗi chi nhánh tự chia tiếp thành các `/64`).

## 5. Quy hoạch và triển khai IPv6 cho doanh nghiệp

```
ISP cấp cho công ty:  2001:db8:1000::/48
                                |
         +----------------------+----------------------+
         |                      |                      |
   Chi nhánh HCM         Chi nhánh Hà Nội      Chi nhánh Đà Nẵng
 /56: ...1000::/56      /56: ...1100::/56      /56: ...1200::/56
         |                      |                      |
     +---+---+              +---+---+              +---+---+
  VLAN10  VLAN20         VLAN10  VLAN20         VLAN10  VLAN20
    /64     /64            /64     /64            /64     /64
```

Trong giai đoạn chuyển đổi (và thực tế hiện nay tại đa số doanh nghiệp), chiến lược phổ biến nhất là **Dual-Stack**: chạy song song cả IPv4 và IPv6 trên cùng hạ tầng, cùng thiết bị — không tắt IPv4 ngay lập tức mà để cả 2 giao thức hoạt động độc lập cho đến khi hạ tầng/ứng dụng đã sẵn sàng hoàn toàn cho IPv6 (đây là con đường mà phần lớn tổ chức lớn, kể cả Google, đang đi qua — thể hiện qua số liệu 50% traffic IPv6 chứ chưa phải 100%).

## 6. Thiết lập IPv6 trên máy tính, thiết bị

- **SLAAC (Stateless Address Autoconfiguration)**: thiết bị tự sinh địa chỉ IPv6 dựa trên prefix router quảng bá (Router Advertisement) cộng với định danh riêng của interface — không cần server DHCP, đơn giản hóa việc cấp phát so với IPv4.
- **DHCPv6**: tương tự DHCP bên IPv4 nhưng dành cho IPv6 — dùng khi cần kiểm soát chặt chẽ hơn (ví dụ gán DNS server cụ thể, ghi log ai nhận IP nào).
- **Lệnh kiểm tra cơ bản**:
  - Windows: `ipconfig` (hiển thị cả IPv6 nếu đã bật), `ping -6 <địa chỉ/domain>`
  - Linux: `ip -6 addr`, `ping6 <địa chỉ/domain>`

## 7. Lab thực hành

**Môi trường**: Cisco Packet Tracer.

1. Bật IPv6 trên 2 Router, gán địa chỉ Global Unicast theo mô hình `/64` cho từng interface LAN.
2. Kiểm tra địa chỉ Link-Local tự động sinh ra trên mỗi interface (bắt đầu bằng `FE80::`) — xác nhận nó tồn tại **song song** với địa chỉ Global Unicast vừa gán tay.
3. Cấu hình định tuyến tĩnh IPv6 cơ bản giữa 2 router (chi tiết cú pháp học ở Bài 13), `ping` xuyên qua để xác nhận thông.
4. Viết tay rút gọn 2-3 địa chỉ IPv6 đầy đủ theo 2 quy tắc ở mục 2, sau đó dùng lệnh `show ipv6 interface` trên router để đối chiếu kết quả router tự hiển thị.

**Kết quả kỳ vọng**: phân biệt được bằng thực hành giữa địa chỉ Link-Local (luôn tồn tại, không cần cấu hình) và Global Unicast (phải gán/cấp phát), đồng thời thành thạo kỹ năng rút gọn địa chỉ tay — kỹ năng bắt buộc trong câu hỏi phỏng vấn.

## 8. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| IPv6 vẫn là công nghệ "tương lai", chưa cần học ngay | Số liệu thực tế cho thấy hơn 50% traffic Internet toàn cầu (Google, 03/2026) đã chạy IPv6 | IPv6 đã là hạ tầng hiện tại, không phải kế hoạch dài hạn |
| IPv6 vẫn có Broadcast giống IPv4, chỉ đổi cách viết địa chỉ | Nhầm lẫn vì cả 2 đều xử lý "gửi cho nhiều thiết bị" | IPv6 loại bỏ hoàn toàn Broadcast, thay bằng Multicast phạm vi hẹp để tránh lãng phí tài nguyên |
| Có thể dùng `::` nhiều lần trong 1 địa chỉ để rút gọn tối đa | Gây địa chỉ mơ hồ, không xác định được số nhóm 0 mỗi cụm đại diện | Chỉ được dùng `::` đúng 1 lần trong toàn bộ địa chỉ |
| Phải tính toán tiết kiệm bit khi chia subnet IPv6 giống VLSM ở IPv4 | Không gian địa chỉ IPv6 quá lớn để cần tiết kiệm ở mức LAN | Chuẩn thực tế luôn dùng `/64` cho mọi subnet LAN thông thường, trọng tâm quy hoạch là tổ chức phân cấp chứ không phải tiết kiệm địa chỉ |

## 9. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Rút gọn địa chỉ `2001:0db8:0000:0000:0000:ff00:0042:8329` theo đúng 2 quy tắc.**
→ Bỏ số 0 đầu mỗi nhóm: `2001:db8:0:0:0:ff00:42:8329`. Thay dãy 3 nhóm 0 liên tiếp bằng `::`: `2001:db8::ff00:42:8329`.

**2. Phân biệt Global Unicast, Link-Local, và Unique Local Address.**
→ Global Unicast (`2000::/3`) định tuyến được trên Internet, tương đương IP Public. Link-Local (`FE80::/10`) chỉ dùng trong cùng 1 đoạn mạng vật lý, tự động sinh ra trên mọi interface. Unique Local (`FC00::/7`) dùng nội bộ tổ chức, tương đương IP Private.

**3. Vì sao IPv6 không còn khái niệm Broadcast?**
→ Vì Broadcast buộc mọi thiết bị xử lý gói tin dù không liên quan, gây lãng phí tài nguyên và nguy cơ broadcast storm. IPv6 thay bằng Multicast phạm vi hẹp (ví dụ `FF02::1` cho tất cả node trên link) — chỉ đúng nhóm thiết bị liên quan mới xử lý.

**4. Vì sao subnet IPv6 hầu như luôn dùng /64 thay vì tính toán tiết kiệm như VLSM ở IPv4?**
→ Không gian địa chỉ IPv6 (128-bit, ~3,4×10³⁸ địa chỉ) lớn đến mức không cần tiết kiệm ở cấp độ LAN — dùng /64 cho mọi subnet là chuẩn thực tế, trọng tâm quy hoạch chuyển sang tổ chức phân cấp (ví dụ /48 → /56 → /64) thay vì tối ưu từng bit.

**5. SLAAC khác DHCPv6 ở điểm nào?**
→ SLAAC thiết bị tự sinh địa chỉ dựa trên prefix router quảng bá, không cần server riêng. DHCPv6 cần server cấp phát tập trung, phù hợp khi cần kiểm soát chặt (ghi log, gán DNS cụ thể) — 2 cơ chế có thể dùng kết hợp trong cùng 1 mạng.

## 10. Tài liệu tham khảo

- [ISOC Pulse — 18 Years Later, IPv6 Reaches Majority](https://pulse.internetsociety.org/en/blog/2026/04/18-years-later-ipv6-reaches-majority/)
- [APNIC Blog — Google hits 50% IPv6](https://blog.apnic.net/2026/04/28/google-hits-50-ipv6/)
- [Google — IPv6 Statistics](https://www.google.com/intl/en/ipv6/statistics.html)
- [NetworkAcademy.IO — IPv6 Address Types](https://www.networkacademy.io/ccna/ipv6/ipv6-address-types)

---
**Liên kết**: [[Bai-05_Dia-chi-IPv4|← Bài 5]] · Bài 7 - Giới thiệu và cấu hình Switch Cisco → (Module 2)
