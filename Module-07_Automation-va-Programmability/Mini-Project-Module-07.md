# Mini Lab — Module 07: Automation và Programmability (CCNA)

> Áp dụng sau khi hoàn thành Bài 29–30.
> File: `Mini-Project-Module-07.md` · Lời giải: `Mini-Project-Module-07-Solution.md`
> Môi trường: Python 3 + Netmiko, Ansible (`pip install netmiko ansible`) + Cisco Packet Tracer (hoặc GNS3 nếu cần SSH thật).

---

## 1. Bối cảnh doanh nghiệp

TechStart JSC vừa ký thêm 1 phòng ban mới — **Marketing** (VLAN 50) — cần onboard vào toàn bộ switch trong 2 tuần tới. Với ~8 thiết bị mạng hiện có (2 router HQ, 3 switch, 2 router chi nhánh/DR, 1 router Partner), sếp IT không muốn đội ngũ ngồi gõ tay từng thiết bị nữa — "1 lần gõ sai lệnh trên switch tầng 2 là cả tầng mất mạng, làm tay mãi không ổn". Bạn được giao dựng 2 công cụ tự động hóa: 1 script Python sao lưu cấu hình + kiểm tra tình trạng thiết bị trước khi đổi, và 1 Ansible playbook đẩy VLAN mới ra đúng những switch cần thiết — có thể chạy lại nhiều lần mà không phá cấu hình đã có.

## 2. Bảng cần điền

**a) Inventory thiết bị** (điền vai trò + có cần VLAN Marketing hay không):

| Thiết bị | Vai trò | Cần VLAN 50 (Marketing)? |
|---|---|---|
| SW-DIST | | |
| SW-ACC1 | | |
| SW-ACC2 | | |
| R-HQ-1, R-HQ-2 | | |
| R-BRANCH, R-DR, R-PARTNER | | |

**b) Phân công công cụ:**

| Công việc | Công cụ đề xuất | Lý do |
|---|---|---|
| Backup running-config toàn bộ thiết bị + kiểm tra SSH còn sống | | |
| Đẩy VLAN 50 + cập nhật allowed-vlan trên trunk, chạy lại được nhiều lần an toàn | | |

## 3. Yêu cầu kỹ thuật

- [ ] Script Python (dùng Netmiko) đọc danh sách thiết bị từ 1 file inventory riêng (không hardcode IP/thiết bị trong code), kết nối SSH tới từng thiết bị, lưu `show running-config` ra file có tên kèm hostname + timestamp.
- [ ] Script Python in ra báo cáo ngắn: thiết bị nào SSH được, thiết bị nào lỗi (không dừng toàn bộ script chỉ vì 1 thiết bị lỗi).
- [ ] Ansible playbook tạo VLAN 50 (tên `MARKETING`) chỉ trên các switch cần (không đụng vào router hay switch không liên quan).
- [ ] Playbook cập nhật `allowed vlan` trên các trunk port liên quan để VLAN 50 lưu thông được giữa các switch.
- [ ] Chạy lại playbook lần 2 không được báo lỗi hay tạo trùng cấu hình (idempotent) — Ansible phải báo `changed=0` ở lần chạy thứ 2 nếu không có gì thay đổi thêm.
- [ ] Thông tin đăng nhập (username/password) không được viết thẳng dạng plain text trong file playbook chính — tách riêng ra file biến/khai báo riêng.

## 4. Ràng buộc

- Không dùng script/playbook để cấu hình VLAN 50 trên switch không có nhu cầu (SW-ACC2 nếu Marketing không ngồi tầng đó, theo bảng inventory đã điền).
- Không được để lộ password dạng plain text khi liệt kê nội dung playbook/script cho người khác xem (che hoặc tách biến).
- Script Python và Ansible playbook phải tách biệt 2 việc rõ ràng theo bảng phân công ở mục 2b — không gộp chung 1 script làm tất cả.

## 5. Tiêu chí đánh giá (Rubric)

| Tiêu chí | Đạt | Chưa đạt |
|---|---|---|
| Script Python | Backup đúng thiết bị, có timestamp, xử lý được lỗi kết nối mà không crash | Hardcode IP trong code, hoặc script dừng hẳn khi 1 thiết bị lỗi |
| Ansible Inventory | Phân nhóm rõ ràng (routers/switches), khớp bảng đã điền | Gộp chung 1 nhóm không phân biệt vai trò |
| Ansible Playbook | Tạo đúng VLAN 50 trên đúng switch, cập nhật đúng trunk | Áp dụng nhầm thiết bị, hoặc thiếu bước allowed-vlan |
| Idempotent | Chạy lại lần 2 báo `changed=0`, không lỗi | Chạy lại báo lỗi hoặc tạo cấu hình trùng lặp |
| Bảo mật credential | Tách riêng file biến, không lộ plain text khi trình bày | Password nằm thẳng trong playbook chính |

## 6. Gợi ý triển khai

1. Bắt đầu từ Bài 29 — dựng inventory file (CSV/YAML đơn giản) trước khi viết logic Netmiko, để script không bị lẫn dữ liệu và code.
2. Với script Python, dùng `try/except` quanh từng kết nối thiết bị — đây là cách chuẩn để 1 thiết bị lỗi không kéo sập toàn bộ vòng lặp.
3. Với Ansible (Bài 30), tách rõ `hosts` trong playbook theo group (`switches`) thay vì `all`, và dùng biến (`vars_files` hoặc Ansible Vault) cho credential.
4. Test idempotent bằng cách chạy playbook 2 lần liên tiếp, so sánh số lượng task báo "changed" giữa 2 lần chạy.

## 7. Lệnh verify cần chạy

- Kiểm tra thư mục backup — đủ số file tương ứng số thiết bị, tên file có timestamp phân biệt được các lần chạy khác nhau.
- `ansible-playbook <playbook>.yml --check` — chạy thử ở chế độ dry-run trước khi áp dụng thật.
- Chạy playbook thật 2 lần liên tiếp, so sánh output `changed=` giữa lần 1 và lần 2.
- `show vlan brief` trên đúng các switch mục tiêu — xác nhận VLAN 50 xuất hiện đúng chỗ, không lan sang thiết bị ngoài phạm vi.

## 8. Kết quả kỳ vọng

Có 1 script Python tự động backup + kiểm tra tình trạng toàn bộ thiết bị mạng trước mỗi lần thay đổi lớn, và 1 Ansible playbook đẩy VLAN Marketing ra đúng switch cần thiết, chạy lại an toàn nhiều lần — đội IT không còn phải gõ tay từng switch mỗi khi có phòng ban mới onboard.

---

**Lưu ý khi viết:**
- Không chèn code hoàn chỉnh (script Python, playbook YAML đầy đủ) vào file đề bài.
- Đúng quy ước "đồ án không chứa câu hỏi/lời giải" — lời giải mẫu để riêng ở `Mini-Project-Module-07-Solution.md`.
- Đề bài buộc dùng kiến thức từ Bài 29 (Netmiko/Python automation) và Bài 30 (Ansible, idempotent, best practice credential).
