# Mini Lab — Module 06: Bảo mật nâng cao (CCNA)

> Áp dụng sau khi hoàn thành Bài 26–28.
> File: `Mini-Project-Module-06.md` · Lời giải: `Mini-Project-Module-06-Solution.md`
> Môi trường: Cisco Packet Tracer (ASA hỗ trợ sẵn trong thư viện thiết bị).

---

## 1. Bối cảnh doanh nghiệp

Đợt audit bảo mật lần 2 của TechStart JSC chỉ ra 3 lỗ hổng nghiêm trọng: (1) access port ở tầng Sales/Kế toán ai cắm switch lạ vào cũng chạy được, không có gì ngăn; (2) mọi router/switch đang dùng chung 1 password local để đăng nhập quản trị — nhân viên IT nghỉ việc vẫn có thể biết password cũ; (3) router biên (R-HQ-1) đang vừa làm router vừa làm "tường lửa" theo kiểu ACL rời rạc, không đủ để chống các kiểu tấn công có chủ đích nhắm vào SRV-WEB đang public ra Internet.

Nhiệm vụ: khóa chặt Lớp 2 ở các access switch, chuyển toàn bộ đăng nhập quản trị sang xác thực tập trung, và đưa 1 thiết bị **Cisco ASA Firewall** thực sự vào làm biên mạng, tách riêng vùng DMZ cho SRV-WEB thay vì để nó nằm chung VLAN nội bộ.

## 2. Sơ đồ & thông tin bổ sung

```
   [ISP-A] ---- outside ---- [ CISCO ASA ] ---- inside ---- [ R-HQ-1 ] ---- (mang noi bo, HSRP...)
                                    |
                                   dmz
                                    |
                              [ SRV-WEB ]   <- chuyen tu VLAN 40 sang day
```

ASA có 3 vùng: `outside` (ra Internet), `inside` (nối R-HQ-1, đại diện toàn bộ mạng nội bộ), `dmz` (chỉ chứa SRV-WEB). Thêm 1 máy chủ AAA mới: **SRV-AAA** (`192.168.10.69`, VLAN 40) chạy TACACS+.

## 3. Bảng cần điền

**a) Security Level Plan (ASA):**

| Interface | Vùng (nameif) | Security level đề xuất | Lý do |
|---|---|---|---|
| Nối ISP-A | | | |
| Nối R-HQ-1 | | | |
| Nối SRV-WEB | | | |

**b) Layer 2 Hardening Plan (SW-ACC1, SW-ACC2):**

| Cơ chế | Áp dụng ở đâu | Mục đích |
|---|---|---|
| Port Security | | |
| DHCP Snooping | | |
| Dynamic ARP Inspection | | |

## 4. Yêu cầu kỹ thuật

- [ ] Mỗi access port trên SW-ACC1/SW-ACC2 chỉ cho phép tối đa 2 địa chỉ MAC (máy tính + điện thoại IP nếu có), học tự động và ghi nhớ qua khởi động lại; vi phạm thì khóa port ở mức hạn chế traffic (không shutdown toàn bộ port).
- [ ] Toàn bộ port nối switch khác (uplink) được tin tưởng cho DHCP; toàn bộ access port không được tin tưởng — chặn được 1 DHCP server giả mạo cắm vào port nhân viên.
- [ ] Dynamic ARP Inspection bật trên các VLAN người dùng, dựa vào bảng DHCP snooping đã có — chặn được tấn công ARP giả mạo (man-in-the-middle) trong cùng VLAN.
- [ ] Toàn bộ router/switch xác thực đăng nhập quản trị qua SRV-AAA (TACACS+); nếu SRV-AAA không phản hồi, vẫn phải đăng nhập được bằng tài khoản local dự phòng (không được khóa cứng luôn cả đội IT).
- [ ] ASA thay thế vai trò router biên trực tiếp ra ISP-A của R-HQ-1 — R-HQ-1 giờ nằm ở vùng `inside`.
- [ ] SRV-WEB chuyển sang vùng `dmz` riêng, không còn nằm trong VLAN 40 nội bộ; Internet chỉ truy cập được đúng cổng 80/443 của SRV-WEB qua NAT tĩnh trên ASA.
- [ ] Partner (đã có ở Module 03) vẫn truy cập được SRV-WEB sau khi đã chuyển sang DMZ.

## 5. Ràng buộc

- Không tắt hoàn toàn port khi vi phạm Port Security (dùng chế độ hạn chế + log, không dùng `shutdown` — tránh phải ra tận nơi bật lại port mỗi lần có sự cố nhỏ).
- SRV-WEB sau khi chuyển sang DMZ, cấu hình NAT/ACL cũ trên R-HQ-1 cho SRV-WEB (từ Module 04) phải được dọn dẹp — không để tồn tại song song 2 nơi NAT cùng 1 dịch vụ.
- Vùng `dmz` không được có security level cao hơn `inside` — phải tuân đúng nguyên tắc phân vùng tin cậy của ASA.
- TACACS+ là phương án chính; local login chỉ là phương án dự phòng, không phải phương án song song ngang hàng.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Port Security | Đúng max-mac, sticky, violation restrict, có log khi vi phạm | Dùng shutdown, hoặc không bật sticky |
| DHCP Snooping/DAI | Trust đúng port uplink, chặn được DHCP giả mạo và ARP giả mạo khi test thử | Trust nhầm access port (mất tác dụng bảo vệ), hoặc bật DAI mà chưa có DHCP snooping binding |
| AAA | Login qua TACACS+ thành công, mất kết nối SRV-AAA vẫn login được bằng local | Không có phương án dự phòng, bị khóa hoàn toàn khi SRV-AAA lỗi |
| ASA Security Level | Đúng outside=0, inside=100, dmz ở giữa, NAT/ACL đúng chiều | Đặt sai mức, hoặc thiếu NAT khiến DMZ không ra được Internet |
| DMZ cho SRV-WEB | Internet chỉ vào được 80/443, Partner vẫn truy cập được sau khi chuyển vùng | Mở rộng quá quyền truy cập, hoặc Partner mất kết nối tới SRV-WEB |

## 7. Gợi ý triển khai

1. Cấu hình Layer 2 hardening trước (Bài 26) — làm trên hạ tầng đã ổn định, dễ test độc lập trước khi động vào phần biên mạng.
2. AAA (Bài 27) nên test kỹ thứ tự phương thức xác thực (`aaa authentication login ... group tacacs+ local`) — thứ tự sai sẽ khiến local không bao giờ được dùng làm dự phòng đúng nghĩa.
3. Khi đưa ASA vào (Bài 28), dựng xong 3 vùng và NAT cơ bản trước, rồi mới dọn dẹp cấu hình cũ trên R-HQ-1 — tránh mất kết nối giữa chừng không biết lỗi ở đâu.
4. Với việc Partner truy cập SRV-WEB sau khi chuyển DMZ, xem lại đường đi thực tế: Partner → R-HQ-1 (đang ở vùng inside) → ASA → DMZ — cần 1 route cụ thể trên R-HQ-1 tới mạng DMZ, không chỉ dựa vào default route.

## 8. Lệnh verify cần chạy

- `show port-security interface` trên SW-ACC1/ACC2 — xác nhận max, sticky MAC, và trạng thái vi phạm nếu có.
- `show ip dhcp snooping` / `show ip dhcp snooping binding` — xác nhận trust đúng port và binding table đã ghi nhận thiết bị thật.
- `show ip arp inspection` — xác nhận DAI đang active trên đúng VLAN.
- Thử đăng nhập vào 1 switch khi tắt SRV-AAA — xác nhận vẫn vào được bằng local.
- Trên ASA: `show nat`, `show access-list`, `packet-tracer` (nếu có) — xác nhận luồng NAT/ACL đúng chiều cho cả traffic từ Internet và từ Partner.

## 9. Kết quả kỳ vọng

Access switch chặn được thiết bị lạ/DHCP giả mạo/ARP giả mạo, toàn bộ đăng nhập quản trị qua xác thực tập trung có dự phòng local, và ASA đảm nhận đúng vai trò tường lửa biên với DMZ tách riêng cho SRV-WEB — Internet chỉ chạm được đúng 80/443, Partner vẫn dùng được dịch vụ như trước.

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (cấu hình Port Security/AAA/ASA cụ thể) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-06-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 26 (Port Security/DHCP Snooping/DAI), Bài 27 (AAA/TACACS+), Bài 28 (Cisco ASA — security level, NAT, ACL, DMZ).
