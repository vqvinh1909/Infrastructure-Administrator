# Lab Tổng Hợp CCNA 04 — Troubleshooting: VLAN/Trunk — LỜI GIẢI MẪU

> File đề bài: `Lab-Tonghop-CCNA-04-Troubleshooting-VLAN-Trunk.md`

---

## 1. Nguyên nhân gốc rễ (root cause) — có 2 lỗi độc lập

1. **VLAN 30 chưa từng được tạo trong VLAN database cục bộ của SW-EDGE-NEW.** Lệnh `switchport access vlan 30` được "copy" từ switch cũ vẫn chấp nhận cú pháp, nhưng vì VLAN 30 không tồn tại trên switch này (điều kiện của bài: SW-EDGE-NEW mới ở chế độ VTP client, hoặc VLAN 30 chưa được tạo/đã bị xóa — nếu switch ở chế độ server/transparent thì IOS sẽ tự tạo VLAN 30 khi gõ `switchport access vlan 30`), port Fa0/5 rơi vào trạng thái **inactive** (lên `up/up` ở Lớp 1 nhưng không forward traffic ở Lớp 2).
2. **Trunk Fa0/1 thiếu VLAN 30 trong danh sách `allowed vlan`** (`10,20,40` — thiếu 30). Dù VLAN 30 có được tạo lại, traffic VLAN 30 vẫn không qua được trunk lên SW-CORE nếu không sửa luôn chỗ này.

## 2. Các bước chẩn đoán dẫn tới kết luận

```
SW-EDGE-NEW# show interfaces status
Port      Status       Vlan       Duplex  Speed Type
Fa0/5     inactive     30   auto   auto  10/100BaseTX
```
→ Trạng thái `inactive` là dấu hiệu rõ ràng nhất: port được gán VLAN 30 (Lớp 1 vẫn tốt, dây vẫn cắm) nhưng VLAN đó không tồn tại trong database của switch nên port không forward traffic.

```
SW-EDGE-NEW# show vlan brief
VLAN Name    Status    Ports
1    default  active   Fa0/2, Fa0/3, Fa0/4, Fa0/6...
```
→ Xác nhận VLAN 30 hoàn toàn không xuất hiện — ngoài VLAN 1 chỉ có các VLAN dành riêng 1002–1005 (mặc định của IOS), đúng như ticket mô tả.

```
SW-EDGE-NEW# show interfaces trunk
Port   Mode   Encapsulation  Status    Native vlan
Fa0/1  on     802.1q         trunking  1

Port   Vlans allowed on trunk
Fa0/1  10,20,40
```
→ Xác nhận lỗi thứ 2 — dù đã sửa lỗi 1, VLAN 30 vẫn không qua được trunk nếu không sửa luôn allowed-vlan.

## 3. Cấu hình sửa lỗi

```
SW-EDGE-NEW(config)# vlan 30
SW-EDGE-NEW(config-vlan)# name KETOAN
SW-EDGE-NEW(config-vlan)# exit

SW-EDGE-NEW(config)# interface fa0/1
SW-EDGE-NEW(config-if)# switchport trunk allowed vlan add 30
```
Dùng `allowed vlan add 30` thay vì viết lại toàn bộ danh sách (`allowed vlan 10,20,30,40`) để tránh rủi ro gõ nhầm làm mất luôn VLAN đang hoạt động tốt (10, 20, 40).

## 4. Output verify sau khi sửa

```
SW-EDGE-NEW# show vlan brief
VLAN Name     Status    Ports
30   KETOAN   active    Fa0/5

SW-EDGE-NEW# show interfaces status
Fa0/5     connected    30         a-full  a-100 10/100BaseTX

SW-EDGE-NEW# show interfaces trunk
Fa0/1  Vlans allowed on trunk: 10,20,30,40
```
PC Kế toán sau đó nhận IP DHCP bình thường và ping được ra ngoài.

## 5. Bài học / phòng tránh tái diễn

- **"Copy y nguyên cấu hình" không đồng nghĩa "mạng hoạt động y hệt"** khi công ty chủ động không dùng VTP Server (quyết định từ Mini Project Module 02) — mỗi switch quản lý VLAN database độc lập, switch thay mới thường chỉ có các VLAN mặc định (1, 1002–1005) trong database, nên phải tạo lại VLAN thủ công dù access port đã trỏ đúng VLAN ID.
- Nên đưa **"tạo đủ VLAN cần thiết + kiểm tra `show vlan brief`"** thành bước bắt buộc đầu tiên trong checklist thay switch, thay vì chỉ copy phần interface config.
- Trạng thái `inactive` trong `show interfaces status` là dấu hiệu chẩn đoán rất đặc trưng cho đúng loại lỗi này — nên nhớ để nhận diện nhanh lần sau, không cần dò từng dòng cấu hình.
- Đây cũng là lý do vì sao 1 số tổ chức chọn dùng VTP (Server/Client) thay vì Transparent — đánh đổi giữa tiện lợi đồng bộ VLAN tự động và rủi ro 1 lỗi VTP lan ra toàn mạng; công ty này đã chọn Transparent (an toàn hơn nhưng đòi hỏi kỷ luật thao tác cao hơn khi thay thiết bị).
