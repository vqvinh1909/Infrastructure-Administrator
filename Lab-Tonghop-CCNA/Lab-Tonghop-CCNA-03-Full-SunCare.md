# Lab Tổng Hợp CCNA 03 — Full: Hạ tầng mạng SunCare Hospital Group (Module 01–07)

> 1 topology duy nhất, Task 1→N phủ đủ 7 module — kịch bản bệnh viện, nhấn mạnh dữ liệu nhạy cảm (hồ sơ bệnh án) và thiết bị y tế quan trọng. Khác 2 lab trước: dùng **GLBP** (thay vì HSRP/VRRP), **OSPF có xác thực MD5**, và **2 SSID Wifi** trên cùng 1 WLC.
> File: `Lab-Tonghop-CCNA-03-Full-SunCare.md` · Lời giải: `Lab-Tonghop-CCNA-03-Full-SunCare-Solution.md`
> Môi trường: Cisco Packet Tracer + Python/Ansible.

---

## Bối cảnh

**SunCare Hospital Group** — bệnh viện có khu Ngoại trú (NGOAITRU), khu Nội trú (NOITRU — thiết bị theo dõi bệnh nhân, không được gián đoạn và không cần ra Internet), IT_SERVER (chứa hệ thống hồ sơ bệnh án điện tử — dữ liệu nhạy cảm nhất, không public), Wifi khách/bệnh nhân (GUEST), và 1 phòng khám vệ tinh (Satellite Clinic) kết nối OSPF. Bệnh viện vừa ra mắt cổng đặt lịch khám trực tuyến cần public an toàn.

## Sơ đồ mạng (đề bài)

```
Internet: [ISP-A] (duong chinh) va [ISP-B] (du phong) noi vao R-HQ-1
          [ISP-C] (duong rieng) noi vao ASA-SC cho vung DMZ (Portal dat lich)

[ R-HQ-1 ]  (GLBP AVG cho VLAN NOITRU, OSPF co xac thuc, NAT, VPN, WAN failover)
      |
[ SW-DIST-SC ]
      |
      |------ [ WLC-SC ] --- AP --- SSID SUNCARE-STAFF (VLAN NGOAITRU)
      |                        `--- SSID SUNCARE-GUEST (VLAN GUEST)
      |
      || (EtherChannel LACP, 2 day du phong)
      |
      |------ [ SW-ACC-NGOAITRU ]  (access port: VLAN NGOAITRU)
      |
      |------ [ SW-ACC-NOITRU ]    (access port: VLAN NOITRU - thiet bi theo doi benh nhan)

[ R-HQ-2 ]  (GLBP AVF phu) cung noi vao SW-DIST-SC

[ ASA-SC ] --- inside --- noi vao SW-DIST-SC (VLAN IT_SERVER)
      |
     dmz
      |
[ SRV-PORTAL ]  (cong dat lich, public qua ISP-C, chi HTTPS)

[ R-HQ-1 ] --- OSPF area 0, MD5 auth --- [ R-CLINIC ]  LAN Phong kham ve tinh
```

## Bảng địa chỉ (cho trước)

| Đối tượng | Địa chỉ | Ghi chú |
|---|---|---|
| VLAN 10 — NGOAITRU | `192.168.50.0/27` | R-HQ-1 `.1`, R-HQ-2 `.2` |
| VLAN 20 — NOITRU | `192.168.50.32/27` | R-HQ-1 `.33`, R-HQ-2 `.34`, GLBP VIP `.35` — **không NAT, không DHCP** |
| VLAN 30 — IT_SERVER | `192.168.50.64/29` | R-HQ-1 `.65`; SRV-AAA `.67`; ASA inside `.66`; SRV-SYSLOG `.69`; WLC mgmt `.68` — **không NAT trực tiếp** |
| VLAN 40 — GUEST | `192.168.50.96/27` | Gateway `.97` |
| WAN HQ↔Clinic | `192.168.50.72/30` | R-HQ-1 `.73`, R-CLINIC `.74` |
| WAN HQ↔ISP-A | `192.168.50.76/30` | R-HQ-1 `.77`, ISP-A `.78` |
| WAN HQ↔ISP-B (dự phòng) | `198.51.100.8/30` | R-HQ-1 `Se0/0/2` `.9`, ISP-B `.10` |
| LAN Clinic | `192.168.51.0/28` | Gateway `.1` |
| VPN Pool (remote) | `192.168.60.0/24` | Chỉ vào NGOAITRU |
| ASA outside (ISP-C) | `198.18.10.0/30` | ASA `.1`, ISP-C `.2` |
| DMZ (SRV-PORTAL) | `192.168.61.0/24` | SRV-PORTAL `.10` |

---

## Task 1 — Chọn phương tiện truyền dẫn

SW-ACC-NOITRU đặt cạnh phòng máy MRI (nhiễu điện từ cực mạnh), cách SW-DIST-SC 60m. Chọn cáp, giải thích (Bài 03).

## Task 2 — IPv6

Chia 1 subnet `/64` từ `2001:db8:sun::/48` (tự thay `sun` bằng 4 hex hợp lệ như Lab 02 — `sun` không phải ký tự hex) cho VLAN NGOAITRU.

## Task 3 — VLAN & Trunk

Tạo VLAN 10, 20, 30, 40. Trunk giữa các switch, Native VLAN đổi sang 999.

## Task 4 — EtherChannel

2 dây dự phòng giữa SW-DIST-SC và SW-ACC-NGOAITRU gộp LACP.

## Task 5 — STP Root Bridge

SW-DIST-SC luôn là Root cho VLAN 10, 20, 30, 40.

## Task 6 — Router-on-a-stick

R-HQ-1 định tuyến đủ 4 VLAN.

## Task 7 — OSPF có xác thực

R-HQ-1 ↔ R-CLINIC chạy OSPF area 0, bật xác thực MD5 trên đoạn kết nối — nếu key sai, neighbor không được lên.

**Kết quả kiểm tra:**
```
R-CLINIC# show ip ospf neighbor
Neighbor ID   State   Address        Interface
1.1.1.1       FULL/-  192.168.50.73  Serial0/0/0

R-CLINIC# show ip ospf interface serial0/0/0 | include authentication
  Message digest authentication enabled
```

## Task 8 — Default route

R-HQ-1 có default route tới ISP-A, quảng bá lại vào OSPF để Clinic tự học.

## Task 9 — DHCP (chỉ NGOAITRU và GUEST)

VLAN NOITRU và IT_SERVER dùng IP tĩnh — không cấu hình DHCP cho 2 VLAN này.

## Task 10 — NAT (loại trừ NOITRU và IT_SERVER)

Chỉ NGOAITRU, GUEST, và Clinic ra Internet qua NAT overload. **NOITRU và IT_SERVER không NAT — không có đường ra Internet.**

**Kết quả kiểm tra:**
```
PC (VLAN NOITRU)> ping 8.8.8.8      -> Request timed out
Server (VLAN IT_SERVER)> ping 8.8.8.8 -> Request timed out
PC (VLAN NGOAITRU)> ping 8.8.8.8    -> Reply
```

## Task 11 — ACL cách ly

- GUEST không được truy cập bất kỳ VLAN nội bộ nào khác (chỉ ra Internet).
- NOITRU ↔ NGOAITRU cách ly 2 chiều.

## Task 12 — NTP & Syslog

R-HQ-1 làm NTP master; toàn bộ thiết bị (kể cả NOITRU dù không ra Internet) gửi log về SRV-SYSLOG qua đường nội bộ.

## Task 13 — GLBP

VLAN NOITRU dùng GLBP (không phải HSRP/VRRP) — cả 2 router cùng chủ động chuyển tiếp traffic (load-balance), không chỉ 1 router active tại 1 thời điểm.

**Kết quả kiểm tra:**
```
R-HQ-1# show glbp brief
Interface  Grp  Fwd Pri State    Address        Active router   Standby router
Gi0/0.20   10   -   110 Active   192.168.50.35  local           192.168.50.34
Gi0/0.20   10   1   -   Active   0007.b400.0a01 local           -
Gi0/0.20   10   2   -   Listen   0007.b400.0a02 192.168.50.34   -
```

## Task 14 — WAN Failover

R-HQ-1 có thêm ISP-B, dùng IP SLA + tracking, bảo vệ đường ra Internet của người dùng/LAN nội bộ (NGOAITRU, GUEST, Clinic) đi qua R-HQ-1. Cổng đặt lịch đi riêng qua ASA-SC + ISP-C nên không thuộc phạm vi failover ISP-A/ISP-B này.

## Task 15 — VPN Remote Access

Bác sĩ trực từ xa kết nối VPN vào R-HQ-1, chỉ truy cập được VLAN NGOAITRU (không được với tới NOITRU/IT_SERVER).

## Task 16 — Wireless / WLC (2 SSID)

WLC-SC phát 2 SSID: `SUNCARE-STAFF` (VLAN NGOAITRU, nhân viên) và `SUNCARE-GUEST` (VLAN GUEST, bệnh nhân/khách) — 2 pre-shared key khác nhau hoàn toàn.

## Task 17 — Layer 2 hardening

- SW-ACC-NGOAITRU: Port Security max 2, sticky, violation restrict.
- SW-ACC-NOITRU: Port Security max **1**, sticky, violation **shutdown** (thiết bị y tế cố định, 1 port = đúng 1 thiết bị).
- DHCP Snooping + DAI trên VLAN NGOAITRU (VLAN GUEST chỉ đi qua Wifi/WLC, không có cổng có dây trên SW-ACC-NGOAITRU nên không áp dụng trên switch này).

## Task 18 — AAA

Đăng nhập thiết bị qua TACACS+ (`SRV-AAA`, `.67`), có local dự phòng.

## Task 19 — ASA & DMZ

SRV-PORTAL (`192.168.61.10`) sau ASA-SC, dùng ISP-C riêng, Internet chỉ vào được cổng **443** (không mở 80 — bắt buộc HTTPS cho dữ liệu y tế).

## Task 20 — Automation

Ansible playbook triển khai nhanh cấu hình khung cho 1 phòng khám vệ tinh thứ 2 mới mở. Python script backup toàn bộ thiết bị định kỳ (phục vụ yêu cầu lưu trữ nhật ký tuân thủ của ngành y tế).

---

## Ràng buộc chung

- NOITRU và IT_SERVER không được xuất hiện trong bất kỳ ACL/NAT nào cho phép ra Internet.
- OSPF giữa HQ và Clinic bắt buộc có xác thực — không được để chạy OSPF không xác thực dù "cho tiện".
- Không đổi bảng địa chỉ đã cho.

## Tiêu chí đánh giá (Rubric)

| Nhóm Task | Trọng số |
|---|---|
| Task 1–2 (Nền tảng) | 5% |
| Task 3–5 (Switching) | 15% |
| Task 6–8 (Routing + OSPF auth) | 15% |
| Task 9–12 (Dịch vụ cơ bản, cách ly dữ liệu nhạy cảm) | 20% |
| Task 13–16 (Remote/Wireless) | 20% |
| Task 17–19 (Bảo mật nâng cao) | 15% |
| Task 20 (Automation) | 10% |

---

**Lưu ý khi viết:** không chèn cấu hình/lệnh sửa lỗi hoàn chỉnh vào file đề bài — lời giải mẫu đầy đủ để riêng ở `Lab-Tonghop-CCNA-03-Full-SunCare-Solution.md`.
