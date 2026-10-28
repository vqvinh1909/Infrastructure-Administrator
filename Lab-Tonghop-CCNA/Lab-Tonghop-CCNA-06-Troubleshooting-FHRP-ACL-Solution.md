# Lab Tổng Hợp CCNA 06 — Troubleshooting: HSRP/ACL — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-06-Troubleshooting-FHRP-ACL.md`

---

## 1. Nguyên nhân gốc rễ — 2 lỗi độc lập

**(a) HSRP:** R-HQ-2 cấu hình `standby 10 ip 10.10.0.30` — sai 1 số so với thiết kế thật (`10.10.0.3`). Về bản chất, R-HQ-2 cùng group 10 với R-HQ-1 nhưng khai báo VIP khác (`10.10.0.30`, không máy trạm nào dùng) — IOS ghi log `%HSRP-4-DIFFVIP1` và bỏ qua hello của peer, nên 2 router không hình thành quan hệ Active/Standby (cả hai cùng tự Active, split-brain) và R-HQ-2 không hề "đứng thay" cho VIP thật mà toàn bộ máy trạm đang trỏ gateway vào (`10.10.0.3`). Khi R-HQ-1 (chủ sở hữu thật của `.3`) tắt đi, không còn thiết bị nào trả lời ARP cho `10.10.0.3` nữa — đây là lý do toàn bộ VLAN mất mạng hoàn toàn thay vì failover mượt.

**(b) ACL:** Cisco IOS xử lý ACL tuần tự, dừng ở dòng đầu tiên khớp điều kiện. Dòng `deny ip any any log` đã nằm **trước** 2 dòng `permit tcp ... eq 80/443` mới thêm — mọi traffic từ Partner (bao gồm cả traffic hợp lệ tới cổng 80/443) bị chặn ngay tại dòng `deny` phía trên, 2 dòng `permit` phía dưới không bao giờ được xét tới dù cú pháp hoàn toàn đúng.

## 2. Các bước chẩn đoán dẫn tới kết luận

```
R-HQ-1# show standby brief
Interface   Grp  Pri  P State   Active     Standby    Virtual IP
Gi0/0.10    10   110  P Active  local      unknown    10.10.0.3

R-HQ-2# show standby brief
Interface   Grp  Pri  P State   Active     Standby    Virtual IP
Gi0/0.10    10   100    Active  local      unknown    10.10.0.30
```
→ 2 router cùng group 10 (cùng vMAC) nhưng lệch Virtual IP nên IOS bỏ qua hello của nhau (log `%HSRP-4-DIFFVIP1`) — cả hai cùng báo `Active`, Standby `unknown` (split-brain). Đây là bằng chứng rõ ràng nhất, dễ bỏ sót nếu chỉ nhìn priority mà không đối chiếu cột Virtual IP.

```
R-HQ-1# show access-lists PARTNER_IN
Extended IP access list PARTNER_IN
    10 permit eigrp host 172.16.1.10 host 224.0.0.10 (140 matches)
    20 permit eigrp host 172.16.1.10 host 172.16.1.9 (112 matches)
    30 deny ip any any log (26 matches)
    40 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq www (0 matches)
    50 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 443 (0 matches)
```
→ Dòng 30 (`deny any any`) có match tăng liên tục khi Partner thử truy cập web; 2 dòng permit bên dưới `0 matches` — xác nhận traffic bị chặn trước khi tới được 2 dòng đó.

## 3. Cấu hình sửa lỗi

**(a) HSRP — R-HQ-2:**
```
interface GigabitEthernet0/0.10
 no standby 10 ip 10.10.0.30
 standby 10 ip 10.10.0.3
```

**(b) ACL — R-HQ-1** (xóa và chèn lại đúng thứ tự, dùng số thứ tự dòng có sẵn của IOS):
```
ip access-list extended PARTNER_IN
 no 30
 60 deny ip any any log
! --- KHONG lam nhu 2 dong duoi day: seq 40 va 50 da ton tai san -> loi 'Duplicate sequence number' ---
 40 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 443
 50 deny ip any any log
```
Dùng số thứ tự dòng (sequence number) để chèn đúng vị trí thay vì xóa nguyên ACL rồi gõ lại toàn bộ — vừa an toàn hơn, vừa đúng ràng buộc đề bài. Lưu ý: ACL hỏng đã có sẵn dòng 40 (`eq www`) và 50 (`eq 443`) nằm sau dòng deny, nên chỉ cần `no 30` rồi thêm lại `deny ip any any log` với số thứ tự lớn hơn (60); gõ lại 40/50 sẽ báo trùng sequence.

## 4. Output verify sau khi sửa

```
R-HQ-1# show standby brief
Gi0/0.10  10  110  P Active   local        10.10.0.2     10.10.0.3

! Tat thu R-HQ-1:
R-HQ-2# show standby brief
Gi0/0.10  10  100    Active   local        unknown              10.10.0.3
! -> R-HQ-2 tiep quan dung VIP that, PC van ping ra ngoai lien tuc khong gian doan
```

```
R-HQ-1# show access-lists PARTNER_IN
    40 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq www (14 matches)
    50 permit tcp 192.168.40.0 0.0.0.255 host 192.168.100.10 eq 443 (9 matches)
    60 deny ip any any log (26 matches)
```

## 5. Bài học / phòng tránh tái diễn

- **Với HSRP/FHRP, luôn đối chiếu Virtual IP giữa 2 router, không chỉ đối chiếu priority** — 1 lỗi gõ nhầm số trong VIP (thường xảy ra khi copy-paste giữa nhiều router có địa chỉ gần giống nhau) khiến toàn bộ redundancy trở thành "vô hình" — trông như đã cấu hình HSRP đầy đủ nhưng thực chất không có 2 router nào thực sự thay thế nhau.
- **`show standby brief` là lệnh nên chạy trên CẢ 2 router cùng lúc** khi nghi ngờ HSRP — chỉ xem 1 phía dễ bỏ sót vì mỗi router đều tự báo cáo trạng thái "Active" của chính nó một cách hợp lệ (dù thực chất cả hai cùng Active vì lệch VIP).
- **Nguyên tắc "top-down, first-match" của ACL Cisco phải được ghi nhớ mỗi khi chỉnh sửa ACL đang có sẵn `deny any any`** — thêm rule mới luôn phải chèn *trước* dòng deny cuối, không phải thêm vào cuối file theo thói quen; đây là lỗi vận hành cực kỳ phổ biến kể cả với kỹ sư có kinh nghiệm khi thao tác vội trong lúc xử lý sự cố.
- **Bộ đếm match trong `show access-lists` là công cụ chẩn đoán ACL đáng tin cậy nhất** — đọc cấu hình bằng mắt dễ bỏ sót thứ tự dòng, nhưng số match tăng ở đúng dòng nào sẽ chỉ thẳng ra traffic thực sự đang "dính" luật nào.
