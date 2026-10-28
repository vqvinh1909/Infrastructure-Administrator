# Lab Tổng Hợp CCNA 05 — Troubleshooting: OSPF/NAT — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-05-Troubleshooting-OSPF-NAT.md`

---

## 1. Nguyên nhân gốc rễ — 2 lỗi độc lập, nối tiếp nhau

1. **R-HQ thiếu `default-information originate` trong tiến trình OSPF.** R-HQ có static default route cục bộ (`ip route 0.0.0.0 0.0.0.0 10.10.0.82`) nhưng không có lệnh này thì OSPF sẽ **không bao giờ** quảng bá route đó cho neighbor, dù neighbor đang Full — đây chính là lý do Store B "thấy" HQ nhưng không có đường ra ngoài.
2. **Access-list 1 dùng cho NAT overload chỉ khớp `10.10.0.0/24`** (wildcard `0.0.0.255` chỉ cho phép octet thứ 4 thay đổi, octet thứ 3 phải đúng `.0`). LAN Store B là `10.10.2.0/28` — khác octet thứ 3 (`.2` thay vì `.0`) nên **không nằm trong access-list**, dù sau khi sửa lỗi 1 và có route, traffic Store B đến `Serial0/0/1` vẫn bị bỏ qua bước NAT, gói tin mang địa chỉ nguồn private ra thẳng Internet và bị rớt.

## 2. Các bước chẩn đoán dẫn tới kết luận

```
R-STOREB# show ip route | include 0.0.0.0
! (khong co dong nao) -> xac nhan Store B chua co default route, du neighbor Full
```

```
R-HQ# show run | section router ospf
router ospf 1
 network 10.10.0.0 0.0.0.31 area 0
 network 10.10.0.72 0.0.0.3 area 0
 network 10.10.0.76 0.0.0.3 area 0
! khong thay dong "default-information originate"
```
→ Xác nhận lỗi 1.

Sau khi thêm `default-information originate` và kiểm tra lại, Store B đã có `O*E2 0.0.0.0/0` — nhưng `ping 8.8.8.8` vẫn timeout:

```
R-HQ# show access-lists 1
Standard IP access list 1
    10 permit 10.10.0.0, wildcard bits 0.0.0.255 (0 match)
```
0 match dù đã có traffic từ Store B đi qua — dấu hiệu rõ ràng access-list không khớp được dải nguồn thực tế.

## 3. Cấu hình sửa lỗi

**Lỗi 1 — R-HQ:**
```
router ospf 1
 default-information originate
```

**Lỗi 2 — R-HQ:**
```
ip access-list standard 1
 permit 10.10.2.0 0.0.0.15
```
Giữ nguyên dòng 10 (HQ `10.10.0.0/24`) và chỉ thêm đúng 1 dòng cho LAN Store B (`10.10.2.0/28`, wildcard `0.0.0.15`) — sửa tối thiểu, không mở rộng phạm vi NAT hơn mức cần thiết, vẫn giữ nguyên cơ chế NAT overload. Dùng dạng `ip access-list standard 1` thay vì `no access-list 1 permit ...` vì trên IOS cổ điển, lệnh `no access-list 1 ...` sẽ xóa cả access-list chứ không chỉ 1 dòng.

## 4. Output verify sau khi sửa

```
R-STOREB# show ip route | include 0.0.0.0
O*E2 0.0.0.0/0 [110/1] via 10.10.0.77, 00:00:05, Serial0/0/0

R-STOREB# ping 8.8.8.8
Success rate is 100 percent (5/5)

R-HQ# show ip nat translations
! Inside global la IP Serial0/0/1 cua HQ (10.10.0.81, link ISP-A 10.10.0.80/30), khong phai 203.0.113.1
Pro Inside global         Inside local        Outside local   Outside global
icmp 10.10.0.81:1         10.10.2.5:1         8.8.8.8:1       8.8.8.8:1
```

## 5. Bài học / phòng tránh tái diễn

- **"Neighbor Full" chỉ chứng minh 2 router trao đổi được LSA** — không chứng minh 1 route cụ thể (đặc biệt route mặc định, vốn phải được quảng bá tường minh bằng `default-information originate`, không tự động như các route nội bộ khác) đã thực sự có trong bảng định tuyến. Đây là ngộ nhận rất phổ biến khi mới học OSPF.
- **Access-list dùng cho NAT cần được rà soát lại mỗi khi thêm site mới** — access-list chỉ khai báo 1 dải `/24` của HQ (không phủ các LAN Store theo kế hoạch địa chỉ) rất dễ "quên" site mới, là lỗi vận hành thực tế cực kỳ phổ biến trong doanh nghiệp đang mở rộng nhanh.
- **`0 match` trong `show access-lists` là tín hiệu chẩn đoán mạnh** — nếu 1 access-list được tham chiếu bởi NAT/ACL mà counter không tăng dù chắc chắn có traffic đi qua, gần như chắc chắn access-list đó không khớp đúng traffic thực tế.
