# Mini Lab — Module 03: Routing (CCNA)

> Áp dụng sau khi hoàn thành Bài 11–15.
> File: `Mini-Project-Module-03.md` · Lời giải: `Mini-Project-Module-03-Solution.md`
> Môi trường: Cisco Packet Tracer.

---

## 1. Bối cảnh doanh nghiệp

TechStart JSC (HQ, đã dựng VLAN/Switching ở Module 02) vừa mở thêm 1 **chi nhánh Q7** và 1 **site dự phòng (DR site)** để backup dữ liệu, đồng thời vừa **mua lại 1 công ty nhỏ (Partner)** dùng hạ tầng router cũ chỉ hỗ trợ EIGRP. Sếp giao nhiệm vụ: dựng toàn bộ định tuyến để 3 site chính (HQ, Chi nhánh Q7, DR) đi lại được với nhau qua giao thức định tuyến động, HQ ra Internet qua 1 đường mặc định, và HQ bắt tay được với router cũ của Partner mà không cần đập đi xây lại thiết bị họ.

## 2. Sơ đồ mạng (đề bài)

```
                [ R-ISP ] -- Se0/0/1 (203.0.113.0/30)
                                |
VLAN 10-40 <--- Gi0/0.x --- [ R-HQ ] ----|--- Se0/1/0 (172.16.1.0/30) --- [ R-BRANCH ]   LAN Q7 (192.168.20.0/24)
  (qua SW-DIST tu Module 2)              |--- Se0/1/1 (172.16.1.4/30) --- [ R-DR ]       LAN DR (192.168.30.0/24)
                                         |--- Se0/2/0 (172.16.1.8/30) --- [ R-PARTNER ]  LAN rieng (192.168.40.0/24,
                                                                                          chi noi EIGRP voi R-HQ)
```

## 3. Bảng cần điền

**a) Địa chỉ WAN (point-to-point, mỗi link 1 khối `/30` riêng — tự chọn dải, không trùng nhau và không trùng LAN đã có ở Module 01):**

| Link | Mạng /30 | IP đầu A | IP đầu B |
|---|---|---|---|
| R-HQ ↔ R-ISP | | | |
| R-HQ ↔ R-BRANCH | | | |
| R-HQ ↔ R-DR | | | |
| R-HQ ↔ R-PARTNER | | | |

**b) Chọn giao thức định tuyến cho từng đoạn:**

| Đoạn | Giao thức đề xuất | Lý do |
|---|---|---|
| R-HQ ↔ R-BRANCH ↔ R-DR (nội bộ công ty) | | |
| R-HQ → R-ISP (ra Internet) | | |
| R-HQ ↔ R-PARTNER (router cũ, chỉ hỗ trợ EIGRP) | | |

## 4. Yêu cầu kỹ thuật

- [ ] R-HQ định tuyến được vào 4 VLAN nội bộ (10/20/30/40 từ Module 02) bằng router-on-a-stick (subinterface trên `Gi0/0`, mỗi subinterface gắn đúng VLAN + encapsulation dot1Q).
- [ ] R-HQ, R-BRANCH, R-DR chạy chung 1 giao thức định tuyến động, quảng bá đủ toàn bộ LAN + WAN liên quan — từ Branch ping được vào máy ở VLAN Sales của HQ, và ngược lại.
- [ ] R-HQ có 1 default route tới R-ISP, và default route này phải được các site Branch/DR học được tự động (không cấu hình static route thủ công trên từng router).
- [ ] R-HQ ↔ R-PARTNER chạy EIGRP riêng (autonomous system riêng nếu cần), 2 bên thấy nhau là neighbor, ping được từ LAN Partner sang LAN HQ (ping xa hơn — sang Branch/DR — không bắt buộc trong bài này).
- [ ] Toàn bộ interface WAN đặt đúng bandwidth/clock rate hợp lý trong Packet Tracer (DCE/DTE) để giao thức định tuyến hoạt động đúng metric.

## 5. Ràng buộc

- Không được cấu hình static route cho từng subnet LAN riêng lẻ giữa HQ/Branch/DR — bài toán yêu cầu dùng giao thức định tuyến động cho việc này (static route chỉ dùng đúng 1 chỗ: default route ra Internet).
- Kết nối tới R-PARTNER **không bắt buộc** phải định tuyến được sang Branch/DR — chỉ cần R-HQ liên lạc 2 chiều với Partner là đủ (không yêu cầu redistribute giữa 2 giao thức định tuyến khác nhau trong bài này).
- Không đặt toàn bộ router chung 1 process ID / AS number giống hệt nhau một cách tùy tiện — phải giải thích được ý nghĩa con số đã chọn.

## 6. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Router-on-a-stick | Đủ 4 subinterface, đúng VLAN/encapsulation, ping được từ router tới từng VLAN | Thiếu subinterface, sai VLAN, hoặc quên `no shutdown` interface vật lý |
| Định tuyến động nội bộ | HQ/Branch/DR học đủ route của nhau, ping xuyên site thành công | Thiếu route ở 1 site, hoặc chỉ ping được 1 chiều |
| Default route ra Internet | Branch/DR tự học được default route qua HQ, không cấu hình tay | Chỉ HQ có default route, Branch/DR không ra được Internet |
| EIGRP với Partner | Neighbor lên đúng, ping 2 chiều HQ↔Partner thành công | Neighbor không lên (sai AS/subnet), hoặc thiếu `network` statement |
| Giải thích lựa chọn | Lý giải đúng vì sao chọn giao thức/metric cho từng đoạn | Chọn ngẫu nhiên không giải thích được sự khác biệt OSPF/EIGRP |

## 7. Gợi ý triển khai

1. Xem lại Bài 12 trước khi cấu hình router-on-a-stick — chú ý interface vật lý `Gi0/0` không được gán IP trực tiếp khi dùng subinterface.
2. Xem lại Bài 14 mục quảng bá default route (lệnh `default-information originate`) — đây là cách để Branch/DR học default route tự động thay vì cấu hình tay từng router.
3. Với EIGRP (Bài 15), nhớ AS number phải khớp giữa R-HQ và R-PARTNER thì neighbor mới lên được — 2 bên không cần dùng cùng AS với vùng OSPF nội bộ.
4. Dựng và test từng đoạn một: LAN nội bộ trước, rồi WAN nội bộ, rồi default route, cuối cùng mới tới EIGRP với Partner — tránh gỡ lỗi nhiều thứ cùng lúc.

## 8. Lệnh verify cần chạy

- `show ip interface brief` trên toàn bộ router — không có interface nào `down/down` ngoài ý muốn.
- `show ip route` trên R-BRANCH và R-DR — phải thấy route tới toàn bộ VLAN của HQ và route `0.0.0.0/0`.
- `show ip protocols` — xác nhận đúng process/AS đang chạy trên từng router.
- `show ip eigrp neighbors` trên R-HQ và R-PARTNER — neighbor phải xuất hiện đúng.
- `ping` từ máy VLAN Sales (HQ) tới máy LAN Branch, DR, và Partner — kiểm chứng bằng chính traffic thật, không chỉ dựa vào bảng định tuyến.

## 9. Kết quả kỳ vọng

R-HQ định tuyến đầy đủ vào 4 VLAN nội bộ, 3 site HQ/Branch/DR trao đổi route động và ping xuyên site được cả 2 chiều, Branch/DR tự học default route ra Internet qua HQ mà không cần cấu hình tay, và R-HQ bắt tay EIGRP thành công với router cũ của Partner — toàn bộ xác minh bằng `show` command và ping thực tế, không chỉ "cấu hình xong".

---

**Lưu ý khi viết:**
- Không chèn đáp án chi tiết (dải IP WAN cụ thể, cấu hình OSPF/EIGRP) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-03-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 12 (Inter-VLAN Routing), Bài 13 (Static default route), Bài 14 (OSPF + default-information originate), Bài 15 (EIGRP).
