# Mini Lab — Module 03: Routing (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-03.md`

---

## 1. Bảng địa chỉ WAN hoàn chỉnh

| Link | Mạng /30 | IP đầu A | IP đầu B |
|---|---|---|---|
| R-HQ ↔ R-ISP | 203.0.113.0/30 | R-HQ: .1 | R-ISP: .2 |
| R-HQ ↔ R-BRANCH | 172.16.1.0/30 | R-HQ: .1 | R-BRANCH: .2 |
| R-HQ ↔ R-DR | 172.16.1.4/30 | R-HQ: 172.16.1.5 | R-DR: 172.16.1.6 |
| R-HQ ↔ R-PARTNER | 172.16.1.8/30 | R-HQ: 172.16.1.9 | R-PARTNER: 172.16.1.10 |

LAN các site: Branch `192.168.20.0/24`, DR `192.168.30.0/24`, Partner `192.168.40.0/24`.

## 2. Chọn giao thức — quyết định

| Đoạn | Giao thức | Lý do |
|---|---|---|
| HQ ↔ Branch ↔ DR | **OSPF area 0**, process 1 | Cả 3 site đều là hạ tầng công ty tự quản, cần hội tụ nhanh và mở rộng thêm site dễ dàng; OSPF là chuẩn mở phổ biến nhất cho mạng doanh nghiệp vừa. |
| HQ → ISP | **Static default route** + `default-information originate` vào OSPF | Không cần biết toàn bộ bảng định tuyến Internet — chỉ cần 1 điểm ra duy nhất; quảng bá lại vào OSPF để Branch/DR tự học mà không cấu hình tay từng router. |
| HQ ↔ Partner | **EIGRP AS 100** | Router cũ của Partner chỉ hỗ trợ EIGRP — không có lựa chọn khác; giữ EIGRP tách biệt khỏi OSPF nội bộ (không redistribute) vì Partner không cần thấy Branch/DR. |

## 3. Cấu hình mẫu

### R-HQ

```
interface GigabitEthernet0/0
 no ip address
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.224
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.10.49 255.255.255.240
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.10.33 255.255.255.240
interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.10.65 255.255.255.240

interface Serial0/0/1
 ip address 203.0.113.1 255.255.255.252
 no shutdown
interface Serial0/1/0
 ip address 172.16.1.1 255.255.255.252
 bandwidth 128
 clock rate 128000
 no shutdown
interface Serial0/1/1
 ip address 172.16.1.5 255.255.255.252
 bandwidth 128
 clock rate 128000
 no shutdown
interface Serial0/2/0
 ip address 172.16.1.9 255.255.255.252
 bandwidth 128
 clock rate 128000
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.0.113.2

router ospf 1
 network 192.168.10.0 0.0.0.31 area 0
 network 192.168.10.32 0.0.0.15 area 0
 network 192.168.10.48 0.0.0.15 area 0
 network 192.168.10.64 0.0.0.15 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.1.4 0.0.0.3 area 0
 default-information originate

! EIGRP trên R-HQ phải quảng bá cả 4 subnet VLAN để Partner ping được tới HQ (và ngược lại)
router eigrp 100
 network 192.168.10.0 0.0.0.31
 network 192.168.10.32 0.0.0.15
 network 192.168.10.48 0.0.0.15
 network 192.168.10.64 0.0.0.15
! (các lệnh network 172.16.1.8 và no auto-summary ở stanza ngay dưới thuộc cùng tiến trình EIGRP 100)

router eigrp 100
 network 172.16.1.8 0.0.0.3
 no auto-summary
```

### R-BRANCH

```
interface Serial0/0/0
 ip address 172.16.1.2 255.255.255.252
 bandwidth 128
 no shutdown
interface GigabitEthernet0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown

router ospf 1
 network 192.168.20.0 0.0.0.255 area 0
 network 172.16.1.0 0.0.0.3 area 0
```

### R-DR

```
interface Serial0/0/0
 ip address 172.16.1.6 255.255.255.252
 bandwidth 128
 no shutdown
interface GigabitEthernet0/0
 ip address 192.168.30.1 255.255.255.0
 no shutdown

router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
 network 172.16.1.4 0.0.0.3 area 0
```

### R-PARTNER

```
interface Serial0/0/0
 ip address 172.16.1.10 255.255.255.252
 bandwidth 128
 no shutdown
interface GigabitEthernet0/0
 ip address 192.168.40.1 255.255.255.0
 no shutdown

router eigrp 100
 network 172.16.1.8 0.0.0.3
 network 192.168.40.0 0.0.0.255
 no auto-summary
```

## 4. Output verify mẫu

```
R-BRANCH# show ip route
O    192.168.10.0/27 [110/782] via 172.16.1.1
O    192.168.30.0/24 [110/1563] via 172.16.1.1
O*E2 0.0.0.0/0 [110/1] via 172.16.1.1
```
Branch học được VLAN Sales của HQ, LAN của DR, và default route ra Internet — tất cả tự động qua OSPF, đúng yêu cầu không cấu hình static route tay.

> **Lưu ý cập nhật:** Cost OSPF tính theo công thức 100000 / bandwidth(kbps). Với `bandwidth 128` trên mọi serial WAN, mỗi hop = 781, cộng thêm 1 của cổng Gi trên R-HQ: Branch → VLAN Sales = 781 + 1 = 782; Branch → LAN DR = 781 + 781 + 1 = 1563. Nếu để bandwidth mặc định của serial (1544 kbps) thì cost mỗi hop là 64 và các giá trị sẽ là [110/65] và [110/129]. Đồng thời R-HQ cần quảng bá các subnet VLAN (192.168.10.x) vào EIGRP 100, nếu không Partner không có route về HQ và ping Partner ↔ HQ sẽ thất bại.

```
R-HQ# show ip eigrp neighbors
H   Address         Interface   Hold  Uptime
0   172.16.1.10     Se0/2/0       13   00:02:41
```

## 5. Giải thích các điểm quan trọng / dễ sai

- **Interface vật lý `Gi0/0` trên R-HQ không được gán IP** khi dùng router-on-a-stick — chỉ subinterface mới gán IP; quên `no shutdown` interface vật lý là lỗi khiến toàn bộ subinterface không hoạt động dù cấu hình đúng.
- **Wildcard mask trong OSPF dễ tính sai** — với `/27` (255.255.255.224) wildcard tương ứng là `0.0.0.31`, không phải `0.0.0.24`; nên tính bằng cách lấy `255 - octet mask` cho từng octet.
- **`default-information originate` chỉ hoạt động nếu R-HQ đã có sẵn 1 default route trong bảng định tuyến** (ở đây là static route tới ISP) — thiếu bước static route trước thì lệnh này không quảng bá được gì.
- **EIGRP AS number phải khớp 2 đầu** (ở đây là 100) — khác AS thì neighbor không bao giờ lên, dù 2 router có cùng nằm trên 1 subnet.
- **EIGRP và OSPF trên R-HQ hoạt động độc lập, không tự trao đổi route** — đây là chủ đích của bài (không yêu cầu redistribute), nên đừng ngạc nhiên khi Branch `show ip route` không thấy `192.168.40.0/24` của Partner.
