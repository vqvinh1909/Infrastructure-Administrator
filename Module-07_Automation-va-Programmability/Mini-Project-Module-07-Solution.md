# Mini Lab — Module 07: Automation và Programmability (CCNA) — LỜI GIẢI MẪU

> File đề bài: `Mini-Project-Module-07.md`

---

## 1. Inventory thiết bị hoàn chỉnh

| Thiết bị | Vai trò | Cần VLAN 50 (Marketing)? |
|---|---|---|
| SW-DIST | Switch trung tâm | Có — cần trunk VLAN 50 đi qua |
| SW-ACC1 | Switch tầng 1 (Sales/Guest — Marketing ngồi chung tầng) | Có — access port thật cho Marketing |
| SW-ACC2 | Switch tầng 2 (Kế toán/NS) | Không — Marketing không ngồi tầng này |
| R-HQ-1, R-HQ-2 | Router | Có — Module 03 dùng router-on-a-stick nên VLAN 50 cần subinterface `Gi0/0.50` (`encapsulation dot1Q 50`), HSRP group, DHCP pool và `network` OSPF; trunk SW-DIST → router cũng phải allow VLAN 50 |
| R-BRANCH, R-DR, R-PARTNER | Router site khác | Không |

*Ghi chú:* toàn bộ thiết bị được giả định có sẵn 1 địa chỉ quản lý trong mạng out-of-band riêng `10.10.10.0/24` (tách biệt khỏi mạng dữ liệu) — thực hành chuẩn doanh nghiệp để quản trị không phụ thuộc vào chính mạng đang cấu hình.

## 2. Phân công công cụ

| Công việc | Công cụ | Lý do |
|---|---|---|
| Backup config + kiểm tra sống/chết toàn bộ 8 thiết bị | **Python + Netmiko** | Cần logic tùy biến (try/except từng thiết bị, đặt tên file theo timestamp) — phù hợp viết script hơn là khai báo trạng thái. |
| Đẩy VLAN 50 + allowed-vlan, chạy lại an toàn | **Ansible** | Đây là bài toán khai báo trạng thái mong muốn ("VLAN 50 phải tồn tại") — đúng sở trường idempotent của Ansible, không cần tự viết logic kiểm tra đã tồn tại hay chưa. |

## 3. Script Python — `backup_devices.py`

```python
import csv
import os
import datetime
from getpass import getpass
from netmiko import ConnectHandler

INVENTORY_FILE = "inventory.csv"
BACKUP_DIR = "backups"


def load_inventory(path):
    with open(path, newline="") as f:
        return list(csv.DictReader(f))


def backup_device(device, username, password):
    conn_params = {
        "device_type": device["device_type"],
        "host": device["ip"],
        "username": username,
        "password": password,
    }
    try:
        os.makedirs(BACKUP_DIR, exist_ok=True)
        conn = ConnectHandler(**conn_params)
        try:
            output = conn.send_command("show running-config")
            ts = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
            filename = f"{BACKUP_DIR}/{device['hostname']}_{ts}.cfg"
            with open(filename, "w") as f:
                f.write(output)
        finally:
            conn.disconnect()
        return device["hostname"], "OK", filename
    except Exception as e:
        return device["hostname"], "FAIL", str(e)


def main():
    username = input("SSH username: ")
    password = getpass("SSH password: ")
    devices = load_inventory(INVENTORY_FILE)

    results = [backup_device(d, username, password) for d in devices]

    print(f"{'Device':<12}{'Status':<8}{'Detail'}")
    for hostname, status, detail in results:
        print(f"{hostname:<12}{status:<8}{detail}")


if __name__ == "__main__":
    main()
```

`inventory.csv`:
```
hostname,ip,device_type
SW-DIST,10.10.10.11,cisco_ios
SW-ACC1,10.10.10.12,cisco_ios
SW-ACC2,10.10.10.13,cisco_ios
R-HQ-1,10.10.10.21,cisco_ios
R-HQ-2,10.10.10.22,cisco_ios
R-BRANCH,10.10.10.23,cisco_ios
R-DR,10.10.10.24,cisco_ios
R-PARTNER,10.10.10.25,cisco_ios
```

Không hardcode username/password trong code — nhập qua `input()`/`getpass()` mỗi lần chạy. 1 thiết bị lỗi kết nối vẫn nằm gọn trong `try/except`, không làm dừng vòng lặp các thiết bị còn lại.

## 4. Ansible — Inventory & Playbook

`inventory.ini`:
```ini
[switches]
sw-dist ansible_host=10.10.10.11
sw-acc1 ansible_host=10.10.10.12
sw-acc2 ansible_host=10.10.10.13

[routers]
r-hq-1 ansible_host=10.10.10.21
r-hq-2 ansible_host=10.10.10.22
r-branch ansible_host=10.10.10.23
r-dr ansible_host=10.10.10.24
r-partner ansible_host=10.10.10.25

[marketing_switches]
sw-dist
sw-acc1

[network:children]
switches
routers

[network:vars]
ansible_network_os=cisco.ios.ios
ansible_become=yes
ansible_become_method=enable
ansible_become_password="{{ vault_enable_password }}"
ansible_connection=network_cli
ansible_user="{{ vault_ssh_user }}"
ansible_password="{{ vault_ssh_password }}"
```

`group_vars/all/vault.yml` (mã hóa bằng `ansible-vault encrypt`):
```yaml
vault_ssh_user: netadmin
vault_ssh_password: SecurePass2026
vault_enable_password: EnablePass2026
```

`host_vars/sw-dist.yml` / `host_vars/sw-acc1.yml`:
```yaml
marketing_trunk_if: Port-channel1
```

`deploy_marketing_vlan.yml`:
```yaml
---
- name: Trien khai VLAN 50 MARKETING len switch can thiet
  hosts: marketing_switches
  gather_facts: no
  tasks:
    - name: Tao VLAN 50 MARKETING
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 50
            name: MARKETING
        state: merged

    - name: Cap nhat allowed-vlan tren trunk uplink
      cisco.ios.ios_l2_interfaces:
        config:
          - name: "{{ marketing_trunk_if }}"
            trunk:
              allowed_vlans: "10,20,30,40,50,999"
        state: merged
```

Chạy: `ansible-playbook -i inventory.ini deploy_marketing_vlan.yml --ask-vault-pass`

> **Lưu ý cập nhật:** playbook này chỉ cấu hình switch. Vì R-HQ-1/R-HQ-2 là router-on-a-stick, trunk từ SW-DIST xuống router cũng phải cho phép VLAN 50 (và router cần subinterface `Gi0/0.50` như mục 1) — nếu không, VLAN 50 không có gateway và không ra khỏi VLAN được.

## 5. Output verify mẫu

```
$ python backup_devices.py
SSH username: netadmin
SSH password:
Device      Status  Detail
SW-DIST     OK      backups/SW-DIST_20260726-143210.cfg
SW-ACC1     OK      backups/SW-ACC1_20260726-143214.cfg
R-PARTNER   FAIL    NetmikoTimeoutException: TCP connection to device failed.
...
```

```
$ ansible-playbook -i inventory.ini deploy_marketing_vlan.yml --ask-vault-pass
PLAY RECAP
sw-acc1     : ok=2  changed=2  ...
sw-dist     : ok=2  changed=2  ...

# Chay lai lan 2:
PLAY RECAP
sw-acc1     : ok=2  changed=0  ...
sw-dist     : ok=2  changed=0  ...
```

## 6. Giải thích các điểm quan trọng / dễ sai

- **Dùng module chính thức `cisco.ios.ios_vlans` / `cisco.ios.ios_l2_interfaces` thay vì `ios_config` với danh sách lệnh thô** — module chính thức tự kiểm tra trạng thái hiện tại trước khi đổi (idempotent thật sự); dùng `ios_config` với `lines:` dễ báo `changed=yes` ở mọi lần chạy dù không có gì thay đổi.
- **`hosts: marketing_switches`, không phải `hosts: switches`** — đây chính là cách đảm bảo SW-ACC2 không bị đụng vào, khớp đúng ràng buộc đề bài.
- **Ansible Vault không phải là "ẩn password đi cho đỡ ai nhìn thấy"** — nó mã hóa thật sự file biến, để file đó commit vào Git chung với team vẫn an toàn, khác với việc chỉ tách file `.gitignore` (vẫn lộ nếu ai đó có file).
- **Script Python và Ansible không nên làm chung 1 việc** — cố dùng Ansible để "backup config" cũng được (`ios_command` + `copy`), nhưng logic linh hoạt như "tiếp tục nếu 1 thiết bị lỗi, in báo cáo dạng bảng tùy biến" viết bằng Python tự nhiên và dễ maintain hơn nhiều so với ép vào playbook.
