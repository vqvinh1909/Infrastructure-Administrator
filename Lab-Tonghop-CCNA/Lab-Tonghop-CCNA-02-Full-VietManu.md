# Lab Tổng Hợp CCNA 02 — Full: Hạ tầng mạng VietManu Manufacturing (Module 01–07)

> 1 topology duy nhất, Task 1→N phủ đủ 7 module — kịch bản nhà máy sản xuất, nhấn mạnh cách ly OT/IT (khác GreenMart Retail ở Lab 01: dùng EIGRP thay vì OSPF, VRRP thay vì HSRP, và có 1 site chỉ dùng static route).
> File: `Lab-Tonghop-CCNA-02-Full-VietManu.md` · Lời giải: `Lab-Tonghop-CCNA-02-Full-VietManu-Solution.md`
> Môi trường: Cisco Packet Tracer + Python/Ansible.

---

## Bối cảnh

**VietManu Manufacturing** — nhà máy sản xuất có khu Văn phòng (VanPhong), khu Sản xuất (SanXuat — chạy PLC/SCADA, theo chuẩn an ninh OT phải cách ly tuyệt đối khỏi mạng văn phòng và **không được ra Internet**), 1 Kho nguyên liệu (Warehouse) kết nối bằng EIGRP, và 1 Xưởng vệ tinh nhỏ (Workshop) chỉ có 1 đường truyền duy nhất nên dùng static route thay vì giao thức động. Công ty vừa đưa vào vận hành 1 cổng ERP nội bộ cần public ra Internet cho đối tác cung ứng.

## Sơ đồ mạng (đề bài)

```
Internet: [ISP-A] (duong chinh) va [ISP-B] (du phong) noi vao R-HQ-1
          [ISP-C] (duong rieng) noi vao ASA-VM cho vung DMZ (ERP)

[ R-HQ-1 ]  (VRRP Master VLAN VanPhong, EIGRP, static toi Workshop, NAT, VPN, WAN failover)
      |
[ SW-DIST-VM ]
      |
      |------ [ WLC-VM ] --- AP --- SSID VIETMANU-STAFF-WIFI (VLAN VanPhong)
      |
      || (EtherChannel LACP, 2 day du phong)
      |
      |------ [ SW-ACC-VANPHONG ]  (access port: VLAN VanPhong)
      |
      |------ [ SW-ACC-SANXUAT ]   (access port: VLAN SanXuat - PLC/SCADA, KHONG NAT)

[ R-HQ-2 ]  (VRRP Backup) cung noi vao SW-DIST-VM

[ ASA-VM ] --- inside --- noi vao SW-DIST-VM (VLAN IT_SERVER)
      |
     dmz
      |
[ SRV-ERP ]  (public qua ISP-C, NAT tinh)

[ R-HQ-1 ] --- EIGRP AS 200 --- [ R-WAREHOUSE ]  LAN Kho nguyen lieu
[ R-HQ-1 ] --- static route (1 duong duy nhat) --- [ R-WORKSHOP ]  LAN Xuong ve tinh
```

## Bảng địa chỉ (cho trước)

| Đối tượng | Địa chỉ | Ghi chú |
|---|---|---|
| VLAN 10 — VANPHONG | `172.20.0.0/27` | R-HQ-1 `.1`, R-HQ-2 `.2`, VRRP VIP `.3` |
| VLAN 20 — SANXUAT | `172.20.0.32/27` | Gateway `.33` — **không NAT, không DHCP** |
| VLAN 30 — IT_SERVER | `172.20.0.64/29` | R-HQ-1 `.65`; SRV-AAA `.67`; ASA inside `.66`; SRV-SYSLOG `.69`; WLC mgmt `.68` |
| WAN HQ↔Workshop | `172.20.0.72/30` | R-HQ-1 `.73`, R-WORKSHOP `.74` |
| WAN HQ↔Warehouse | `172.20.0.76/30` | R-HQ-1 `.77`, R-WAREHOUSE `.78` |
| WAN HQ↔ISP-A | `172.20.0.80/30` | R-HQ-1 `.81`, ISP-A `.82` |
| WAN HQ↔ISP-B (dự phòng) | `198.51.100.4/30` | R-HQ-1 `Se0/0/2` `.5`, ISP-B `.6` |
| LAN Workshop | `172.20.1.0/28` | Gateway `.1` |
| LAN Warehouse | `172.20.2.0/28` | Gateway `.1` |
| VPN Pool (remote) | `172.20.50.0/24` | Chỉ vào được VLAN VanPhong |
| ASA outside (ISP-C) | `200.150.60.0/30` | ASA `.1`, ISP-C `.2` |
| DMZ (SRV-ERP) | `172.21.0.0/24` | SRV-ERP `.10` |

---

## Task 1 — Chọn phương tiện truyền dẫn

SW-DIST-VM đặt tại phòng kỹ thuật; SW-ACC-SANXUAT đặt cạnh khu vực máy hàn/động cơ công suất lớn (nhiễu điện rất mạnh), cách 70m. Chọn loại cáp và giải thích (Bài 03).

## Task 2 — IPv6

Chia 1 subnet `/64` từ `2001:db8:vman::/48` (tự thay `vman` bằng 4 hex bất kỳ hợp lệ) cho VLAN VANPHONG, gán trên subinterface R-HQ-1.

## Task 3 — VLAN & Trunk

Tạo VLAN 10 (VANPHONG), 20 (SANXUAT), 30 (IT_SERVER). Trunk giữa SW-DIST-VM và 2 switch access, đổi Native VLAN sang 999.

**Kết quả kiểm tra:**
```
SW-DIST-VM# show interfaces trunk
Native vlan 999   Vlans allowed: 10,20,30,999
```

## Task 4 — EtherChannel

2 dây dự phòng giữa SW-DIST-VM và SW-ACC-VANPHONG gộp EtherChannel LACP.

## Task 5 — STP Root Bridge

SW-DIST-VM luôn là Root cho VLAN 10, 20, 30.

## Task 6 — Router-on-a-stick

R-HQ-1 định tuyến 3 VLAN, đúng địa chỉ ở bảng trên.

## Task 7 — EIGRP với Warehouse

R-HQ-1 và R-WAREHOUSE chạy EIGRP AS 200, quảng bá đủ LAN/WAN liên quan.

**Kết quả kiểm tra:**
```
R-WAREHOUSE# show ip eigrp neighbors
H  Address        Interface  Hold  Uptime
0  172.20.0.77    Se0/0/0     12   00:03:10
R-WAREHOUSE# show ip route eigrp
D  172.20.0.0/27 [90/...] via 172.20.0.77
```

## Task 8 — Static route với Workshop

R-HQ-1 ↔ R-WORKSHOP chỉ dùng static route (2 chiều) — không chạy EIGRP/OSPF trên đoạn này. R-HQ-1 có static route tới LAN Workshop `172.20.1.0/28`; R-WORKSHOP chỉ cần 1 static default route về `172.20.0.73` (đã bao gồm đường về toàn bộ mạng HQ).

**Kết quả kiểm tra:**
```
R-WORKSHOP# show ip route static
S*   0.0.0.0/0 [1/0] via 172.20.0.73
```

## Task 9 — Default route ra Internet cho Warehouse & Workshop

R-HQ-1 có default route tới ISP-A. Warehouse phải tự học default route qua EIGRP (không cấu hình tay); Workshop dùng thêm 1 dòng static default route riêng (vì không chạy EIGRP nên không thể tự học).

**Kết quả kiểm tra:**
```
R-WAREHOUSE# show ip route | include ^D\*EX
D*EX  0.0.0.0/0 [170/...] via 172.20.0.77

R-WORKSHOP# show ip route static | include ^S\*
S*  0.0.0.0/0 [1/0] via 172.20.0.73
```

## Task 10 — DHCP

VLAN VANPHONG nhận IP tự động. **VLAN SANXUAT không cấu hình DHCP** (thiết bị PLC/SCADA dùng IP tĩnh cố định — đúng thực hành an ninh OT).

## Task 11 — NAT (loại trừ SanXuat)

VANPHONG, IT_SERVER, Warehouse, Workshop ra Internet qua NAT overload tại `Serial0/0/1`. **VLAN SANXUAT không được NAT — không có đường ra Internet dưới bất kỳ hình thức nào.**

**Kết quả kiểm tra:**
```
PC (VLAN SANXUAT)> ping 8.8.8.8   -> Request timed out (dung, day la yeu cau bat buoc)
PC (VLAN VANPHONG)> ping 8.8.8.8  -> Reply (dung)
```

## Task 12 — ACL cách ly SanXuat ↔ VanPhong

2 chiều đều bị chặn, cả 2 vẫn hoạt động nội bộ bình thường trong VLAN của mình.

## Task 13 — NTP & Syslog

R-HQ-1 làm NTP master, toàn bộ router gửi log về SRV-SYSLOG (`.69`) — kể cả log từ SanXuat (dù SanXuat không ra Internet, log vẫn phải về được SRV-SYSLOG qua đường nội bộ).

## Task 14 — VRRP

VLAN VANPHONG có 2 gateway (R-HQ-1 Master, R-HQ-2 Backup), VIP `172.20.0.3`. Tắt thử R-HQ-1 không được mất kết nối.

**Kết quả kiểm tra:**
```
R-HQ-2# show vrrp brief
Interface  Grp  Pri  Time  Own  Pre  State    Master addr   Group addr
Gi0/0.10   10   100  3609       Y   Master  172.20.0.2  172.20.0.3   ! sau khi tat R-HQ-1
```

## Task 15 — WAN Failover

R-HQ-1 có thêm ISP-B dự phòng, dùng IP SLA + tracking, bảo vệ đường ra Internet của người dùng/LAN nội bộ (VANPHONG, IT_SERVER, Warehouse, Workshop) đi qua R-HQ-1. Cổng ERP đi riêng qua ASA-VM + ISP-C nên không thuộc phạm vi failover ISP-A/ISP-B này.

## Task 16 — VPN Remote Access

Kỹ sư làm việc từ xa kết nối VPN vào R-HQ-1, chỉ truy cập được VLAN VANPHONG (**không được cấu hình để với tới VLAN SANXUAT** — đúng nguyên tắc cách ly OT).

## Task 17 — Wireless / WLC

WLC-VM quản lý AP tại khu văn phòng, SSID `VIETMANU-STAFF-WIFI` ánh xạ VLAN VANPHONG.

## Task 18 — Layer 2 hardening (khác biệt giữa VanPhong và SanXuat)

- SW-ACC-VANPHONG: Port Security max 2 MAC, sticky, violation **restrict**.
- SW-ACC-SANXUAT: Port Security max **1** MAC, sticky, violation **shutdown** (thiết bị OT cố định, 1 port = đúng 1 thiết bị, phát hiện bất thường phải khóa cứng ngay, không chỉ hạn chế).
- DHCP Snooping + DAI trên VLAN VANPHONG.

**Kết quả kiểm tra:**
```
SW-ACC-SANXUAT# show port-security interface fa0/5
Violation Mode: Shutdown   Maximum MAC: 1
```

## Task 19 — AAA & ASA/DMZ

Đăng nhập thiết bị qua TACACS+ (`SRV-AAA`, `.67`) có local dự phòng. SRV-ERP (`172.21.0.10`) đặt sau ASA-VM, dùng ISP-C riêng, Internet chỉ vào được cổng 443.

## Task 20 — Automation

Ansible playbook triển khai nhanh cấu hình khung (VLAN + IP + static route) cho 1 xưởng vệ tinh mới mở, không đụng Workshop/Warehouse đang chạy. Python script backup toàn bộ thiết bị.

---

## Ràng buộc chung

- VLAN SANXUAT không được xuất hiện trong bất kỳ ACL/NAT/DHCP nào cho phép ra Internet — vi phạm ràng buộc này coi như sai toàn bộ nhóm Task liên quan.
- Workshop chỉ dùng static route — không được bật EIGRP/OSPF trên đoạn này dù có thể "tiện" hơn.
- Không đổi bảng địa chỉ đã cho.

## Tiêu chí đánh giá (Rubric)

| Nhóm Task | Trọng số |
|---|---|
| Task 1–2 (Nền tảng) | 5% |
| Task 3–5 (Switching) | 15% |
| Task 6–9 (Routing: EIGRP + Static) | 20% |
| Task 10–13 (Dịch vụ cơ bản, cách ly OT) | 20% |
| Task 14–17 (Remote/Wireless) | 15% |
| Task 18–19 (Bảo mật nâng cao) | 15% |
| Task 20 (Automation) | 10% |

---

**Lưu ý khi viết:** không chèn cấu hình/lệnh sửa lỗi hoàn chỉnh vào file đề bài — lời giải mẫu đầy đủ để riêng ở `Lab-Tonghop-CCNA-02-Full-VietManu-Solution.md`.
