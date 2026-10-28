---
title: Bài 30 - Network Automation với Python
type: Configuration
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — viết được script Python/Netmiko tự động hóa tác vụ cấu hình cơ bản là kỹ năng thực chiến bậc nhất trong tuyển dụng NetOps/Sysadmin hiện nay, và đây là bài cuối cùng khép lại lộ trình 30 bài Quản trị Hạ tầng mạng"
tags: [networking, automation, python, netmiko, ansible, backup, vlan, sysadmin, tong-ket]
aliases: [Netmiko, Python Network Automation, send_config_set, Ansible Playbook, Idempotent]
related: ["[[Bai-29_Network-Automation-va-Programmability]]"]
last_review: 2026-07-21
---

# Bài 30: Network Automation với Python

> [!important] Tại sao cần file này
> [[Bai-29_Network-Automation-va-Programmability|Bài 29]] đã giới thiệu **tư duy** automation (SDN, API, các công cụ phổ biến bao gồm Ansible/Puppet/Python) — bài cuối cùng này đưa tư duy đó vào thực hành cụ thể với **Python và Netmiko**, công cụ scripting linh hoạt nhất để quản trị thiết bị mạng. Đây cũng là bài khép lại toàn bộ lộ trình 30 bài — phần cuối sẽ tổng kết lại roadmap và hướng đi tiếp theo sau khi hoàn thành khóa học.

## 1. Giới thiệu Python và thư viện Netmiko

**Python** là ngôn ngữ lập trình phổ biến nhất cho Network Automation nhờ cú pháp đơn giản, dễ tiếp cận với người không chuyên lập trình. **Netmiko** là thư viện Python xây dựng trên nền Paramiko (thư viện SSH client cho Python), giúp đơn giản hóa việc kết nối SSH và gửi lệnh tới hàng chục dòng thiết bị mạng khác nhau (Cisco IOS, Juniper, Arista...) mà không cần tự viết logic xử lý SSH từ đầu.

```bash
pip install netmiko
```

So với việc dùng Ansible (mục 7) — nơi quản trị viên mô tả **trạng thái mong muốn** bằng YAML — Netmiko cho phép viết logic **hoàn toàn tùy biến** bằng Python: đọc dữ liệu từ file, xử lý điều kiện rẽ nhánh phức tạp, gọi thêm API khác, hoặc tính toán trước khi quyết định gửi lệnh gì tới thiết bị.

## 2. Kết nối tới thiết bị — khai báo thông tin và thiết lập session

```python
from netmiko import ConnectHandler

thiet_bi = {
    "device_type": "cisco_ios",
    "host": "10.10.10.11",
    "username": "admin",
    "password": "Admin@2026",
    "secret": "Enable@2026",
    "port": 22,
}

ket_noi = ConnectHandler(**thiet_bi)
ket_noi.enable()
print(ket_noi.find_prompt())
ket_noi.disconnect()
```

| Tham số | Ý nghĩa |
|---|---|
| `device_type` | Cho Netmiko biết đây là dòng thiết bị gì (`cisco_ios`, `cisco_asa`, `juniper_junos`...) để xử lý đúng cú pháp prompt/lệnh riêng của từng hãng |
| `host` | Địa chỉ IP quản trị của thiết bị |
| `username`/`password` | Thông tin đăng nhập — nên dùng service account riêng đã nêu ở [[Bai-29_Network-Automation-va-Programmability|Bài 29]] mục 8, không hardcode mật khẩu thật trong script khi dùng ở môi trường thật |
| `secret` | Mật khẩu `enable` (Privileged EXEC), dùng khi gọi `ket_noi.enable()` |
| `port` | Port SSH, mặc định 22 nếu không khai báo |

`ConnectHandler(**thiet_bi)` thiết lập kết nối SSH dựa trên dictionary đã khai báo. `find_prompt()` trả về đúng dấu nhắc dòng lệnh hiện tại của thiết bị (hữu ích để xác nhận đã kết nối đúng và đang ở đúng mode nào). `disconnect()` đóng kết nối — nên luôn gọi ở cuối script để tránh giữ session SSH treo trên thiết bị.

> [!question] Vì sao nên gọi `disconnect()` tường minh ở cuối script thay vì để Python tự dọn dẹp khi chương trình kết thúc?
> Vì thiết bị mạng thường có **giới hạn số lượng session VTY đồng thời** (đã học ở [[Bai-27_AAA|Bài 27]] mục 2, ví dụ `line vty 0 4` chỉ cho tối đa 5 session) — nếu script không đóng kết nối đúng cách (đặc biệt khi script gặp lỗi giữa chừng), session SSH có thể bị treo trên thiết bị cho tới khi hết timeout, dần dần chiếm hết số session VTY khả dụng và khiến cả quản trị viên thật lẫn các script chạy sau đó không thể kết nối được nữa. Đây là lý do thực hành tốt luôn đặt `disconnect()` trong khối `finally` (mục 5) để đảm bảo kết nối luôn được đóng dù script có gặp lỗi hay không.

## 3. Script tự động hóa — Tạo VLAN

```python
from netmiko import ConnectHandler

thiet_bi = {"device_type": "cisco_ios", "host": "10.10.10.11",
            "username": "admin", "password": "Admin@2026"}

lenh_cau_hinh = [
    "vlan 10",
    "name NHANVIEN",
    "vlan 20",
    "name KETOAN",
]

ket_noi = ConnectHandler(**thiet_bi)
ket_qua = ket_noi.send_config_set(lenh_cau_hinh)
print(ket_qua)
ket_noi.disconnect()
```

`send_config_set()` tự động đưa thiết bị vào chế độ `configure terminal`, gửi lần lượt từng lệnh trong danh sách, rồi thoát khỏi chế độ cấu hình — tương đương chính xác những gì quản trị viên gõ tay để tạo 2 VLAN, nhưng có thể lặp lại **giống hệt nhau** trên hàng chục switch chỉ bằng cách lặp (loop) qua danh sách thiết bị.

## 4. Script tự động hóa — Thiết lập định tuyến

```python
lenh_dinh_tuyen = [
    "ip route 0.0.0.0 0.0.0.0 203.0.113.1",
    "router ospf 1",
    "network 10.10.10.0 0.0.0.255 area 0",
]

ket_noi = ConnectHandler(**thiet_bi)
ket_qua = ket_noi.send_config_set(lenh_dinh_tuyen)
print(ket_qua)
ket_noi.disconnect()
```

Cùng dùng `send_config_set()` như mục 3, chỉ khác nội dung lệnh — ở đây khai báo 1 default route ([[Bai-13_Static-Route|Bài 13]]) và bật OSPF cho 1 mạng cụ thể ([[Bai-14_OSPF|Bài 14]]). Điều này cho thấy Netmiko không có khái niệm "module chuyên biệt cho từng loại cấu hình" như Ansible — nó chỉ đơn thuần gửi **đúng chuỗi lệnh CLI** quản trị viên vẫn gõ tay, nên yêu cầu người viết script phải tự nắm chắc cú pháp CLI chính xác (đã học xuyên suốt Bài 7-28).

## 5. Script tự động hóa — Sao lưu cấu hình thiết bị

```python
from netmiko import ConnectHandler
from datetime import datetime

thiet_bi = {"device_type": "cisco_ios", "host": "10.10.10.11",
            "username": "admin", "password": "Admin@2026"}

ket_noi = None
try:
    ket_noi = ConnectHandler(**thiet_bi)
    cau_hinh_hien_tai = ket_noi.send_command("show running-config")

    thoi_gian = datetime.now().strftime("%Y%m%d_%H%M%S")
    ten_file = f"backup_{thiet_bi['host']}_{thoi_gian}.txt"

    with open(ten_file, "w") as f:
        f.write(cau_hinh_hien_tai)

    print(f"Da sao luu thanh cong: {ten_file}")
except Exception as loi:
    print(f"Sao luu that bai: {loi}")
finally:
    if ket_noi:
        ket_noi.disconnect()
```

Script này là 1 trong những ứng dụng Netmiko phổ biến nhất thực tế: `send_command("show running-config")` lấy toàn bộ cấu hình hiện tại của thiết bị, sau đó ghi ra 1 file `.txt` có kèm **timestamp** trong tên file (để không ghi đè bản backup cũ, giữ lại lịch sử theo thời gian) — tương đương thao tác `copy running-config tftp` đã học ở [[Bai-11_Gioi-thieu-ve-Dinh-tuyen|Bài 11]] mục 8, nhưng tự động hóa hoàn toàn và có thể chạy định kỳ (qua cron job hoặc Scheduled Task) cho hàng loạt thiết bị mà không cần con người can thiệp.

> [!question] Vì sao dùng khối `try/except/finally` trong script backup này lại quan trọng hơn hẳn so với script tạo VLAN đơn giản ở mục 3?
> Vì backup cấu hình thường được thiết kế để chạy **định kỳ, không giám sát trực tiếp** (ví dụ mỗi đêm, tự động qua cron job) — không có người ngồi trực tiếp quan sát để can thiệp nếu có lỗi. Nếu 1 thiết bị trong danh sách backup gặp sự cố kết nối (mất mạng, sai mật khẩu, SSH timeout) mà script không được bọc trong `try/except`, toàn bộ script sẽ **dừng đột ngột ngay tại thiết bị lỗi đó** — mọi thiết bị còn lại trong danh sách (nếu backup nhiều thiết bị bằng vòng lặp) sẽ **không được backup luôn**, dù bản thân chúng hoàn toàn không có vấn đề gì. `try/except` cho phép script ghi nhận lỗi của riêng thiết bị đó rồi **tiếp tục** với thiết bị tiếp theo, còn `finally` đảm bảo `disconnect()` luôn được gọi (liên hệ mục 2) dù có lỗi xảy ra hay không — tránh treo session VTY trên thiết bị.

## 6. Giám sát quá trình thực thi script và kiểm tra kết quả đầu ra

Chạy xong script không có nghĩa là mọi thứ đã đúng như mong đợi — cần chủ động kiểm tra kết quả, không chỉ tin tưởng script "chạy xong không báo lỗi":

```python
import logging

logging.basicConfig(filename="automation.log", level=logging.INFO,
                     format="%(asctime)s - %(levelname)s - %(message)s")

ket_noi = None
try:
    ket_noi = ConnectHandler(**thiet_bi)
    ket_qua = ket_noi.send_config_set(lenh_cau_hinh)
    logging.info(f"Cau hinh thanh cong tren {thiet_bi['host']}: {ket_qua}")

    # Kiem tra ket qua dau ra tren thiet bi ngay sau khi cau hinh
    kiem_tra = ket_noi.send_command("show vlan brief")
    if "NHANVIEN" in kiem_tra:
        logging.info(f"Xac nhan VLAN 10 (NHANVIEN) da duoc tao dung tren {thiet_bi['host']}")
    else:
        logging.warning(f"VLAN 10 KHONG xuat hien sau khi cau hinh tren {thiet_bi['host']}")
except Exception as loi:
    logging.error(f"Loi khi xu ly {thiet_bi['host']}: {loi}")
finally:
    if ket_noi:
        ket_noi.disconnect()
```

Module `logging` của Python ghi lại **toàn bộ quá trình thực thi** ra file `automation.log` kèm timestamp — thay vì chỉ `print()` ra màn hình (mất ngay khi đóng terminal), log được lưu lại lâu dài để đối chiếu sau này, tương tự tinh thần Syslog tập trung đã học ở [[Bai-18_NTP-va-Syslog|Bài 18]] và [[Bai-28_Cisco-ASA-Firewall|Bài 28]] mục 9. Quan trọng hơn: script không chỉ dừng lại ở việc **gửi lệnh cấu hình** — nó còn chủ động chạy thêm 1 lệnh `show` để **xác nhận kết quả thực tế trên thiết bị** có đúng như kỳ vọng hay không (`"NHANVIEN" in kiem_tra`), thay vì chỉ giả định lệnh gửi đi ắt hẳn đã thành công.

> [!question] Vì sao chỉ kiểm tra "script chạy không báo lỗi" (không exception) là chưa đủ để khẳng định tác vụ cấu hình đã thành công?
> Vì Netmiko chỉ báo lỗi (raise exception) khi có vấn đề ở tầng **kết nối/giao tiếp** (mất SSH, timeout...); với lệnh cấu hình sai, thiết bị chỉ trả về dòng lỗi như `% Invalid input detected` và `send_config_set()` mặc định KHÔNG raise exception — phải tự kiểm tra output, hoặc truyền `error_pattern=...` (khi đó mới có `ConfigInvalidException`) — nó không tự động hiểu được **ý nghĩa nghiệp vụ** của lệnh vừa gửi có đúng như mong muốn hay không. Ví dụ: gửi đúng cú pháp lệnh tạo VLAN nhưng thiết bị từ chối áp dụng vì đã đạt giới hạn số VLAN tối đa, hoặc gõ nhầm tên VLAN thành `"NHANVIEN "` (thừa dấu cách) — cả 2 trường hợp này Netmiko vẫn "chạy xong không lỗi" vì về mặt kết nối/cú pháp hoàn toàn hợp lệ, nhưng kết quả thực tế trên thiết bị lại không đúng như kỳ vọng. Đây là lý do bước xác nhận kết quả đầu ra (chạy lại lệnh `show` và kiểm tra nội dung) là bước bắt buộc, không thể bỏ qua, đúng tinh thần "không có gì là đã hoàn tất cho tới khi verify được trên chính thiết bị" — nguyên tắc lặp lại xuyên suốt mọi bài Verify & Troubleshoot trong khóa học này.

## 7. Ansible cho Network Automation

Bên cạnh Python/Netmiko (mục 1-6), **Ansible** là công cụ automation phổ biến khác đã nhắc tới ở [[Bai-29_Network-Automation-va-Programmability|Bài 29]] mục 7 — thay vì viết logic từng bước như Python, Ansible mô tả **trạng thái mong muốn** bằng YAML.

```ini
[switches]
sw-access-01 ansible_host=10.10.10.11
sw-access-02 ansible_host=10.10.10.12

[switches:vars]
ansible_network_os=ios
ansible_connection=network_cli
ansible_user=admin
ansible_password=Admin@2026
```

```yaml
---
- name: Cau hinh VLAN tren switch access
  hosts: switches
  gather_facts: no
  tasks:
    - name: Dam bao VLAN 10 ton tai
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 10
            name: NHANVIEN
        state: merged
```

```bash
ansible-playbook -i inventory.ini cau-hinh-vlan.yml
```

Khác biệt cốt lõi so với script Netmiko ở mục 3: Ansible có tính **Idempotent** — chạy lại đúng Playbook này nhiều lần cho **cùng 1 kết quả cuối cùng**. Nếu VLAN 10 đã tồn tại đúng như khai báo, Ansible tự nhận ra và **không làm gì thêm**; trong khi script Netmiko ở mục 3 nếu chạy lại nhiều lần sẽ **gửi lại y nguyên lệnh `vlan 10`** mỗi lần (không gây lỗi vì IOS tự bỏ qua nếu VLAN đã tồn tại, nhưng bản thân script không có khái niệm "kiểm tra trước khi làm" như Ansible).

> [!question] Vậy nên chọn Netmiko/Python hay Ansible cho 1 tác vụ automation cụ thể?
> Ansible phù hợp nhất cho các tác vụ có thể mô tả rõ ràng bằng "trạng thái mong muốn" (declarative) — ví dụ: đảm bảo VLAN/ACL/NTP server tồn tại đúng như khai báo, tận dụng tính Idempotent để chạy an toàn lặp lại nhiều lần. Netmiko/Python phù hợp hơn khi cần **logic điều kiện phức tạp** (imperative) — như script backup có timestamp ở mục 5, hay script cần tự phân tích output rồi ra quyết định tiếp theo (mục 6) — điều Playbook YAML thuần khai báo trạng thái khó biểu diễn gọn gàng. Trong thực tế, nhiều đội NetOps dùng kết hợp cả 2: Ansible cho các tác vụ cấu hình lặp lại có cấu trúc rõ ràng, Python/Netmiko cho các tác vụ cần xử lý logic/dữ liệu phức tạp hơn (như backup, phân tích log, tích hợp với hệ thống khác).

## 8. Verify & Troubleshoot

```
ansible-playbook -i inventory.ini cau-hinh-vlan.yml --check
ansible switches -i inventory.ini -m cisco.ios.ios_command -a "commands='show version'"
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Netmiko báo lỗi timeout khi kết nối | Sai IP/thiếu SSH trên thiết bị đích, hoặc firewall chặn port 22 | Kiểm tra kết nối SSH thủ công trước bằng terminal thông thường để loại trừ nguyên nhân hạ tầng |
| `send_config_set()` chạy xong nhưng cấu hình không đúng như mong đợi | Sai cú pháp lệnh CLI trong danh sách, hoặc thiết bị từ chối 1 phần lệnh (ví dụ do quyền tài khoản) | Luôn thêm bước xác nhận qua lệnh `show` sau khi cấu hình (mục 6), không chỉ tin `send_config_set()` không báo lỗi |
| Script backup dừng giữa chừng khi backup nhiều thiết bị | Thiếu `try/except` bao quanh từng thiết bị, 1 thiết bị lỗi làm dừng toàn bộ vòng lặp | Bọc phần xử lý từng thiết bị trong `try/except` riêng biệt (mục 5), ghi log lỗi rồi tiếp tục thiết bị kế tiếp |
| Playbook chạy xong nhưng không thấy thay đổi nào trên thiết bị | Trạng thái khai báo đã khớp với trạng thái hiện tại (đúng hành vi Idempotent, không phải lỗi) | Chạy với `--check` (chế độ mô phỏng, không áp dụng thật) để xem Ansible dự kiến thay đổi gì trước khi chạy thật |
| Không rõ script/Playbook đang thực hiện bước nào khi gặp lỗi | Thiếu thông tin chi tiết khi chạy | Với Ansible: thêm `-vvv`. Với Python: kiểm tra file log đã cấu hình ở mục 6 |

## 9. Lab thực hành

**Môi trường**: 1 máy Linux/WSL có cài Python + Netmiko + Ansible (`pip install netmiko ansible`), Cisco Packet Tracer hoặc GNS3 với ít nhất 1 switch bật SSH.

1. Viết script Netmiko theo mục 2 để kết nối và in ra `show version` của switch thử nghiệm.
2. Viết script tạo VLAN theo mục 3, chạy và xác nhận bằng `show vlan brief` trực tiếp trên switch.
3. Viết script sao lưu cấu hình theo mục 5 (có `try/except/finally` và timestamp trong tên file), chạy 2 lần liên tiếp và xác nhận có 2 file backup khác timestamp.
4. Thêm phần logging + kiểm tra kết quả đầu ra theo mục 6 vào script tạo VLAN ở bước 2, xác nhận file `automation.log` ghi đúng kết quả xác nhận VLAN.
5. Viết Inventory + Playbook Ansible theo mục 7 cho cùng switch đó, chạy 2 lần liên tiếp và quan sát lần thứ 2 báo `changed=0` — minh chứng tính Idempotent.

**Kết quả kỳ vọng**: tự tay viết được script Python/Netmiko hoàn chỉnh cho 3 tác vụ thực chiến phổ biến nhất (tạo VLAN, thiết lập định tuyến, backup cấu hình), hiểu rõ tầm quan trọng của việc giám sát/xác nhận kết quả thay vì chỉ tin script "chạy xong không lỗi".

## 10. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| Script chạy xong không báo lỗi nghĩa là tác vụ đã thành công đúng ý muốn | Nhầm lẫn giữa "không lỗi kết nối/cú pháp" và "đúng kết quả nghiệp vụ" | Netmiko không hiểu ý nghĩa nghiệp vụ của lệnh — luôn cần xác nhận lại qua lệnh `show` trên chính thiết bị (mục 6) |
| Không cần `try/except` nếu script chỉ chạy cho 1 thiết bị duy nhất, thử nghiệm | Đánh giá thấp rủi ro ngay cả ở quy mô nhỏ | Ngay cả với 1 thiết bị, thiếu `finally: disconnect()` vẫn có thể treo session VTY nếu script lỗi giữa chừng (mục 2, mục 5) |
| Netmiko và Ansible là 2 công cụ cạnh tranh, chỉ nên chọn 1 | Nhìn nhận automation như lựa chọn nhị phân | Trong thực tế thường dùng kết hợp: Ansible cho tác vụ khai báo trạng thái có cấu trúc, Netmiko/Python cho logic phức tạp cần xử lý dữ liệu linh hoạt hơn (mục 7) |
| Học Python/Netmiko automation là thay thế hoàn toàn cho việc hiểu CLI/giao thức mạng | Đánh giá sai vai trò của automation | Automation xây dựng trên nền kiến thức đã học xuyên suốt Bài 7-28 — không hiểu VLAN/OSPF hoạt động thế nào thì không thể viết đúng danh sách lệnh cho `send_config_set()` (mục 3-4) |

## 11. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Netmiko là gì, và giải quyết vấn đề gì khi tự động hóa thiết bị mạng?**
→ Netmiko là thư viện Python xây dựng trên Paramiko, đơn giản hóa việc kết nối SSH và gửi lệnh tới nhiều dòng thiết bị mạng khác nhau — giúp quản trị viên không cần tự viết logic xử lý SSH/prompt từ đầu cho từng hãng thiết bị.

**2. `send_command()` và `send_config_set()` khác nhau ở điểm nào?**
→ `send_command()` dùng để chạy lệnh `show` (không thay đổi cấu hình) và trả về kết quả đã xử lý sẵn. `send_config_set()` tự động đưa thiết bị vào chế độ `configure terminal`, gửi danh sách lệnh cấu hình, rồi thoát khỏi chế độ đó — dùng để thay đổi cấu hình thiết bị.

**3. Vì sao nên dùng `try/except/finally` khi viết script backup cấu hình cho nhiều thiết bị?**
→ Để 1 thiết bị gặp lỗi kết nối không làm dừng toàn bộ vòng lặp backup các thiết bị còn lại (`try/except`), và đảm bảo kết nối SSH luôn được đóng đúng cách dù có lỗi xảy ra hay không, tránh treo session VTY trên thiết bị (`finally`).

**4. Vì sao cần kiểm tra kết quả đầu ra trên thiết bị sau khi script cấu hình xong, thay vì chỉ tin `send_config_set()` không báo lỗi?**
→ Vì Netmiko chỉ báo lỗi ở tầng kết nối (lệnh sai cú pháp mặc định không raise exception mà chỉ nằm trong output, trừ khi đặt `error_pattern`), không hiểu ý nghĩa nghiệp vụ của lệnh — lệnh có thể gửi thành công về mặt cú pháp nhưng bị thiết bị từ chối áp dụng hoặc áp dụng sai ý muốn (ví dụ do giới hạn tài nguyên hoặc gõ nhầm tham số), cần chạy thêm lệnh `show` để xác nhận đúng trạng thái thực tế.

**5. So sánh cách tiếp cận declarative (Ansible) và imperative (Python/Netmiko) trong network automation.**
→ Ansible (declarative) mô tả trạng thái mong muốn, tự động kiểm tra và chỉ áp dụng thay đổi cần thiết (Idempotent). Python/Netmiko (imperative) thực thi tuần tự đúng các lệnh đã viết, linh hoạt cho logic phức tạp nhưng không tự có cơ chế kiểm tra trạng thái trước khi hành động như Ansible.

## 12. Tổng kết khóa học — Lộ trình 30 bài Quản trị Hạ tầng mạng

Nhìn lại toàn bộ 7 module đã đi qua, có thể thấy lộ trình được xây dựng theo đúng cách 1 Sysadmin thực thụ tiếp cận hạ tầng mạng — từ nền tảng tới nâng cao:

| Module | Chủ đề | Bài |
|---|---|---|
| 2 | Switching — VLAN, Trunking, STP, EtherChannel | 7-10 |
| 3 | Routing — Static Route, OSPF, EIGRP | 11-15 |
| 4 | Dịch vụ và bảo mật cơ bản — ACL, NTP/Syslog, DHCP, NAT | 16-20 |
| 5 | Remote Access và Wireless — FHRP, VPN, Load Balancing, Wireless LAN, WLC | 21-25 |
| 6 | Bảo mật nâng cao — Nguy cơ bảo mật, cấu hình bảo mật thiết bị, Firewall ASA | 26-28 |
| 7 | Automation và Programmability — SDN, Python/Netmiko, Ansible | 29-30 |

Từng module đều xây dựng trên nền module trước — VLAN (Module 2) là điều kiện để hiểu Inter-VLAN Routing (Module 3), NAT/ACL (Module 4) là nền tảng để hiểu ASA (Module 6), và toàn bộ kiến thức CLI xuyên suốt Bài 7-28 chính là thứ giúp viết đúng script/Playbook đang tự động hóa **cái gì** (Module 7) — automation không thể thay thế việc hiểu bản chất giao thức, nó chỉ giúp triển khai kiến thức đó nhanh và nhất quán hơn ở quy mô lớn.

**Hướng đi tiếp theo sau khi hoàn thành khóa học**: luyện lab thực tế nhiều hơn (GNS3/EVE-NG cho mô phỏng gần với thiết bị thật hơn Packet Tracer), tìm hiểu sâu hơn 1 trong các hướng chuyên sâu (Security — hướng theo case study Module 6; Automation/DevNet — hướng theo Module 7; hoặc Wireless/Data Center); và quan trọng nhất — chuẩn bị trả lời **câu hỏi thực chiến** cho từng chủ đề đã học (đã có sẵn trong mỗi bài) bằng chính lời của mình, không học thuộc lòng, để thực sự sẵn sàng cho phỏng vấn Sysadmin Junior/Mid-level.

## 13. Tài liệu tham khảo

- [Netmiko — Simplified Network Device Automation in Python](https://netmiko.org/)
- [PyNet Labs — What is Netmiko and how to use it in Python?](https://www.pynetlabs.com/what-is-netmiko-and-how-to-use-it-in-python/)
- [Medium — Network Automation with Python and Netmiko: Backup configs](https://medium.com/@mickaelsoares/network-automation-with-python-and-netmiko-backup-configs-4fd84e957d3f)
- [Network Journey — Cisco Netmiko Scripting with Examples](https://networkjourney.com/cisco-netmiko-scripting-with-examples-a-comprehensive-guide/)
- [Ansible Community Documentation — Network Examples](https://docs.ansible.com/projects/ansible/latest/network/user_guide/network_best_practices_2.5.html)
- [PacketCoders — The Beginner's Guide to the Ansible Inventory](https://www.packetcoders.io/the-beginners-guide-to-the-ansible-inventory/)

---
**Liên kết**: [[Bai-29_Network-Automation-va-Programmability|← Bài 29]] · Hoàn thành khóa học
