---
title: Bài 29 - Giới thiệu về Network Automation
type: Concept
difficulty: Intermediate
lab_required: true
exam: "Sysadmin Mid-level — Network Automation là xu hướng vận hành hạ tầng doanh nghiệp hiện đại, và Automation and Programmability cũng là chủ đề cuối cùng trong blueprint CCNA 200-301"
tags: [networking, automation, sdn, programmability, api, json, yaml, dna-center, ansible, puppet, sysadmin]
aliases: [Network Automation, SDN, Software Defined Networking, Northbound API, Southbound API, REST API, JSON, YAML, Cisco DNA Center, Puppet]
related: ["[[Bai-07_Gioi-thieu-va-cau-hinh-Switch-Cisco]]", "[[Bai-30_Ansible-va-Tong-ket]]"]
last_review: 2026-07-21
---

# Bài 29: Giới thiệu về Network Automation

> [!important] Tại sao cần file này
> Từ Bài 7 tới Bài 28, mọi cấu hình đều thực hiện theo cách **truyền thống**: đăng nhập CLI từng thiết bị, gõ lệnh thủ công. Cách này hoạt động tốt với vài chục thiết bị, nhưng không thể mở rộng khi hạ tầng có hàng trăm, hàng nghìn switch/router/firewall trải khắp nhiều địa điểm — đúng vấn đề mà VTP ([[Bai-09_Trunking-DTP-VTP|Bài 9]]) và WLC ([[Bai-25_Cisco-Wireless-Controller|Bài 25]]) đã từng giải quyết ở quy mô nhỏ hơn. Bài này giới thiệu tư duy **quản trị hạ tầng theo hướng lập trình (Programmability)** — nền tảng của Sysadmin/NetOps hiện đại, và cũng là chủ đề cuối cùng trong bộ đề CCNA 200-301.

## 1. Xu hướng và lợi ích của Network Automation

**Xu hướng**: khi hạ tầng doanh nghiệp mở rộng về quy mô (nhiều chi nhánh, nhiều thiết bị, hạ tầng lai giữa on-premise và cloud), việc cấu hình thủ công từng thiết bị ngày càng trở nên bất khả thi — cả về thời gian lẫn tính nhất quán. Ngành mạng đang dịch chuyển từ mô hình "Network Engineer gõ CLI" sang mô hình **NetOps/DevOps cho hạ tầng mạng** — nơi cấu hình được quản lý như code (Infrastructure as Code), có thể kiểm tra lại (review), lưu lịch sử thay đổi (version control), và triển khai tự động.

**Lợi ích cụ thể**:

| Lợi ích | Giải thích |
|---|---|
| **Giảm sai sót do con người** | Gõ tay hàng trăm lần cùng 1 loại lệnh dễ gõ nhầm; script/Playbook thực thi chính xác như đã viết, không "mệt" hay "lơ đãng" |
| **Tăng tốc độ triển khai** | Áp dụng 1 thay đổi cấu hình lên hàng trăm thiết bị chỉ trong vài phút, thay vì nhiều ngày làm thủ công |
| **Đảm bảo tính nhất quán** | Mọi thiết bị cùng vai trò được cấu hình theo đúng 1 chuẩn (template) giống hệt nhau, tránh tình trạng "mỗi thiết bị 1 kiểu" do nhiều người cấu hình thủ công qua thời gian |
| **Dễ audit và khôi phục** | Cấu hình lưu dưới dạng file (script/Playbook) có thể version control (như Git), biết chính xác ai đổi gì, khi nào, dễ dàng rollback nếu có sự cố |
| **Giải phóng nhân lực cho việc phức tạp hơn** | Tự động hóa các tác vụ lặp lại, để Sysadmin tập trung vào thiết kế kiến trúc, xử lý sự cố phức tạp, và cải tiến hệ thống |

> [!question] Vì sao "tính nhất quán" lại được xem là lợi ích quan trọng ngang với tốc độ, dù nghe có vẻ là lợi ích phụ?
> Vì phần lớn sự cố khó troubleshoot nhất trong hạ tầng mạng thực tế không đến từ lỗi thiết kế, mà từ **sự khác biệt nhỏ, tích lũy qua thời gian** giữa các thiết bị lẽ ra phải giống hệt nhau — ví dụ 1 switch access thiếu đúng 1 dòng cấu hình Port Security ([[Bai-26_Bao-mat-Lop-2|Bài 26]] mục 10) mà không ai để ý, vì nó được cấu hình thủ công bởi 1 người khác vào 1 thời điểm khác. Automation buộc mọi thiết bị cùng vai trò phải khớp chính xác với 1 nguồn cấu hình chuẩn (template/Playbook) duy nhất — loại bỏ hoàn toàn khả năng "trôi cấu hình" (configuration drift) kiểu này, vốn là nguyên nhân gốc rễ của rất nhiều sự cố khó chẩn đoán trong môi trường vận hành thủ công lâu năm.

## 2. Các công nghệ phổ biến trong Network Automation

| Công nghệ | Bản chất | Ví dụ |
|---|---|---|
| **Scripting** | Viết script (thường bằng Python) tự động gửi lệnh/tương tác trực tiếp với thiết bị qua SSH/API | Script Python dùng thư viện Netmiko (chi tiết ở [[Bai-30_Ansible-va-Tong-ket|Bài 30]]) |
| **SDN** (Software-Defined Networking) | Tách Control Plane ra khỏi thiết bị, tập trung vào 1 Controller có cái nhìn toàn cục (chi tiết ở mục 3-4) | Cisco Catalyst Center (trước đây là DNA Center), Cisco ACI |
| **Network Tools/Configuration Management** | Công cụ chuyên dụng mô tả **trạng thái mong muốn** của hạ tầng bằng file khai báo, tự động áp dụng và duy trì đúng trạng thái đó | Ansible, Puppet (chi tiết ở mục 7) |

Ba công nghệ này không loại trừ nhau — trong thực tế thường **kết hợp**: SDN Controller cung cấp API, còn Scripting/Network Tools là công cụ **gọi tới** API đó (hoặc gọi trực tiếp SSH nếu thiết bị không có API) để tự động hóa tác vụ.

## 3. Traditional Networking vs Controller-Based Networking

Mọi thiết bị mạng vận hành dựa trên 3 mặt phẳng (plane) chức năng:

| Mặt phẳng | Nhiệm vụ | Ví dụ |
|---|---|---|
| **Data Plane** | Chuyển tiếp gói tin thực tế qua thiết bị | Switch chuyển frame theo bảng MAC, router chuyển packet theo routing table |
| **Control Plane** | Quyết định **cách** chuyển tiếp — xây dựng bảng định tuyến/MAC | OSPF/EIGRP tính toán route ([[Bai-14_OSPF|Bài 14]], [[Bai-15_EIGRP|Bài 15]]), STP tính toán topology ([[Bai-10_STP-va-EtherChannel|Bài 10]]) |
| **Management Plane** | Cho phép quản trị viên cấu hình/giám sát thiết bị | CLI, SSH, SNMP |

| Mô hình | Vị trí 3 mặt phẳng | Cách quản trị |
|---|---|---|
| **Traditional Networking** | Cả 3 mặt phẳng đều nằm **ngay trên từng thiết bị** | Đăng nhập CLI từng thiết bị, cấu hình thủ công (cách đã làm xuyên suốt Bài 7-28) |
| **Controller-Based Networking (SDN)** | **Control Plane tách riêng**, tập trung tại 1 Controller (theo mô hình SDN thuần; trên thực tế Catalyst Center vẫn giữ Control Plane phân tán trên thiết bị, Controller chủ yếu là lớp quản lý/tự động hóa/policy); Data Plane vẫn ở từng thiết bị, chỉ lo chuyển tiếp gói tin | Quản trị viên tương tác với Controller (qua giao diện/API), Controller tự động đẩy cấu hình xuống toàn bộ thiết bị |

> [!question] Vì sao tách riêng Control Plane khỏi thiết bị lại giúp quản trị hạ tầng lớn dễ dàng hơn nhiều?
> Vì trong mô hình Traditional, mỗi thiết bị **tự tính toán quyết định của riêng nó** dựa trên thông tin cục bộ (Hello packet, LSA...) — muốn thay đổi hành vi toàn mạng, phải sửa cấu hình **từng thiết bị một**. Khi Control Plane được tập trung vào 1 Controller, Controller có **cái nhìn toàn cục** về toàn bộ topology mạng cùng lúc, và có thể tính toán + đẩy quyết định xuống **toàn bộ thiết bị chỉ trong 1 lần thao tác** — tương tự cách WLC ([[Bai-25_Cisco-Wireless-Controller|Bài 25]]) tập trung quản lý hàng trăm AP thay vì cấu hình Autonomous AP từng cái một. Đây chính là lý do Controller-Based Networking mở rộng quy mô tốt hơn nhiều so với mô hình truyền thống.

## 4. Kiến trúc SDN — Northbound API và Southbound API

**SDN** là kiến trúc hiện thực hóa mô hình Controller-Based ở mục 3, với 3 lớp rõ rệt:

```
    +--------------------+
    |    APPLICATIONS    |
    | (Automation tools) |
    +--------------------+
             ^
       | Northbound API (REST/JSON)
             v
    +--------------------+
    |   SDN CONTROLLER   |
    |  (Control Plane)   |
    +--------------------+
             ^
       | Southbound API (NETCONF/OpenFlow)
             v
    +--------------------+
    |  NETWORK DEVICES   |
    |    (Data Plane)    |
    +--------------------+
```

| Loại API | Kết nối giữa | Giao thức/chuẩn phổ biến |
|---|---|---|
| **Northbound API** | Controller ↔ Ứng dụng/công cụ tự động hóa phía trên | REST API (dùng JSON/XML qua HTTPS) |
| **Southbound API** | Controller ↔ Thiết bị mạng thực tế phía dưới | NETCONF, RESTCONF, OpenFlow, SNMP |

Southbound API cho phép Controller **đẩy cấu hình xuống** thiết bị (tương tự việc gõ CLI, nhưng tự động và đồng loạt). Northbound API cho phép **ứng dụng bên ngoài** (dashboard giám sát, script tự động hóa, hệ thống ticket) "hỏi" Controller về trạng thái mạng hoặc "ra lệnh" thay đổi cấu hình, mà không cần biết chi tiết kỹ thuật của từng thiết bị bên dưới.

> **Lưu ý cập nhật:** các khái niệm hay gặp ở Southbound API — **YANG** là ngôn ngữ mô hình dữ liệu mô tả cấu trúc cấu hình/trạng thái của thiết bị; **NETCONF** truyền dữ liệu theo mô hình YANG dạng XML qua SSH (port 830); **RESTCONF** truy cập cùng dữ liệu đó qua HTTPS với JSON hoặc XML. Ngoài GET/POST/PUT/DELETE, RESTCONF còn dùng **PATCH** để cập nhật một phần tài nguyên. Mã trạng thái HTTP thường gặp: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 500 Internal Server Error.

> [!question] Vì sao gọi là "Northbound" (hướng Bắc) và "Southbound" (hướng Nam)?
> Đây là quy ước hình ảnh hóa xuất phát từ cách vẽ sơ đồ kiến trúc SDN theo chiều dọc: Controller luôn được vẽ ở giữa, ứng dụng/công cụ quản trị vẽ **phía trên** (hướng Bắc trên bản đồ theo quy ước), còn thiết bị mạng thực tế vẽ **phía dưới** (hướng Nam). API "đi lên" phía ứng dụng gọi là Northbound, API "đi xuống" phía thiết bị gọi là Southbound — tên gọi thuần túy mang tính quy ước hình ảnh, không phản ánh cơ chế kỹ thuật khác biệt gì ngoài **hướng giao tiếp** trong sơ đồ kiến trúc.

## 5. Định dạng dữ liệu dùng trong Automation — JSON, XML, YAML

REST API (Northbound, mục 4) cần 1 định dạng dữ liệu chuẩn để Controller và ứng dụng "hiểu nhau" khi trao đổi thông tin cấu hình/trạng thái — 3 định dạng phổ biến nhất:

```json
{
  "hostname": "SW-ACCESS-01",
  "vlan": 10,
  "interface": "GigabitEthernet0/1"
}
```

```yaml
hostname: SW-ACCESS-01
vlan: 10
interface: GigabitEthernet0/1
```

```xml
<device>
  <hostname>SW-ACCESS-01</hostname>
  <vlan>10</vlan>
  <interface>GigabitEthernet0/1</interface>
</device>
```

| Định dạng | Đặc điểm | Trường hợp dùng phổ biến |
|---|---|---|
| **JSON** (JavaScript Object Notation) | Gọn nhẹ, dễ đọc bằng máy lẫn con người, cấu trúc key-value | Phổ biến nhất cho REST API hiện đại |
| **XML** (Extensible Markup Language) | Có thẻ mở/đóng rõ ràng, hỗ trợ schema/validation chặt chẽ | Dùng nhiều trong NETCONF (Southbound API) |
| **YAML** (YAML Ain't Markup Language) | Dựa trên thụt lề (indentation), dễ đọc nhất với con người, không có dấu ngoặc rườm rà | Phổ biến cho file cấu hình, đặc biệt là Ansible Playbook ([[Bai-30_Ansible-va-Tong-ket|Bài 30]]) |

> [!question] Cả 3 định dạng đều biểu diễn cùng 1 loại thông tin (như ví dụ trên) — vậy vì sao Ansible lại chọn YAML thay vì JSON hay XML cho Playbook?
> Vì mục tiêu chính của Ansible Playbook không chỉ là "máy đọc được" mà còn phải **con người đọc và viết dễ dàng** — quản trị viên mạng thường không phải lập trình viên chuyên nghiệp, và YAML tối giản cú pháp (dựa vào thụt lề thay vì dấu ngoặc `{}`/`<>` như JSON/XML) giúp file cấu hình trông gần giống ngôn ngữ tự nhiên, dễ đọc/sửa hơn nhiều với người mới tiếp cận automation. JSON vẫn là lựa chọn tối ưu hơn khi 2 hệ thống máy tính trao đổi dữ liệu với nhau qua REST API (không cần con người đọc trực tiếp thường xuyên), còn XML phù hợp khi cần validate cấu trúc dữ liệu nghiêm ngặt (như NETCONF).

## 6. REST API cơ bản

**REST** (Representational State Transfer) là kiểu kiến trúc API phổ biến nhất hiện nay cho Northbound API — hoạt động dựa trên các **HTTP Verb** tiêu chuẩn:

| HTTP Verb | Hành động | Tương đương thao tác CRUD |
|---|---|---|
| **GET** | Lấy thông tin, không thay đổi gì | Read |
| **POST** | Tạo mới 1 tài nguyên | Create |
| **PUT** | Cập nhật toàn bộ 1 tài nguyên đã có | Update |
| **DELETE** | Xóa 1 tài nguyên | Delete |

```
GET https://dnacenter.congty.local/dna/intent/api/v1/network-device
```

Ví dụ trên là 1 lệnh GET gửi tới Controller (ở đây là Cisco Catalyst Center, mục 7; request cần kèm token lấy trước từ `/dna/system/api/v1/auth/token`) để lấy danh sách toàn bộ thiết bị mạng đang được quản lý — kết quả trả về thường ở định dạng JSON (mục 5), sau đó có thể được 1 script tự động phân tích và xử lý tiếp mà không cần con người đăng nhập giao diện xem thủ công.

## 7. Các công cụ triển khai phổ biến

| Công cụ | Loại | Đặc điểm |
|---|---|---|
| **Cisco Catalyst Center** (tên cũ: DNA Center) | SDN Controller thương mại | Nền tảng tập trung của Cisco cho hạ tầng Catalyst — thiết kế, cấu hình, giám sát hàng trăm/nghìn thiết bị qua 1 giao diện, chi tiết ở phần dưới |
| **Ansible** | Configuration Management, agentless | Mô tả trạng thái mong muốn bằng YAML (Playbook), kết nối qua SSH — chi tiết ở [[Bai-30_Ansible-va-Tong-ket|Bài 30]] |
| **Puppet** | Configuration Management, thường agent-based | Tương tự mục tiêu với Ansible (mô tả trạng thái mong muốn), nhưng thường yêu cầu cài **agent** trên thiết bị đích để agent tự kiểm tra và áp dụng cấu hình định kỳ — phổ biến hơn trong quản lý server, ít phổ biến hơn Ansible cho riêng mảng thiết bị mạng |
| **Python & Netmiko** | Scripting, thư viện lập trình | Viết script tùy biến hoàn toàn, linh hoạt xử lý logic phức tạp — chi tiết ở [[Bai-30_Ansible-va-Tong-ket|Bài 30]] |

**Cisco Catalyst Center** hiện thực hóa mô hình controller-based (mục 3-4) cho hạ tầng Catalyst — chủ yếu là lớp quản lý, tự động hóa và policy tập trung (qua SSH, SNMP, NETCONF...), còn Control Plane (OSPF, STP...) vẫn chạy phân tán trên từng thiết bị — cho phép quản trị viên **thiết kế, cấu hình, giám sát** hàng trăm/nghìn thiết bị từ 1 giao diện tập trung, thay vì đăng nhập CLI từng thiết bị như xuyên suốt khóa học này.

| Khả năng của Catalyst Center | Mô tả |
|---|---|
| **Design** | Thiết kế topology, site, IP addressing plan tập trung trước khi triển khai |
| **Provision** | Tự động đẩy cấu hình xuống thiết bị mới (Zero Touch Provisioning — thiết bị tự tải cấu hình khi kết nối lần đầu, không cần cấu hình tay) |
| **Policy** | Áp dụng chính sách bảo mật/QoS đồng loạt trên toàn hạ tầng |
| **Assurance** | Giám sát sức khỏe mạng theo thời gian thực, dùng AI/Machine Learning để dự đoán sự cố trước khi xảy ra |

> [!question] Vì sao Ansible (agentless) lại phổ biến hơn Puppet (thường agent-based) cụ thể trong mảng tự động hóa thiết bị mạng, dù Puppet vẫn rất phổ biến trong quản lý server?
> Vì đa số thiết bị mạng (router, switch, firewall truyền thống) chạy hệ điều hành đóng (như Cisco IOS) — **không có khả năng cài đặt thêm phần mềm agent tùy ý** như 1 server Linux/Windows thông thường có thể làm. Puppet cần agent chạy trên chính thiết bị đích để tự kiểm tra và báo cáo trạng thái về Puppet Master — điều này khó/không khả thi trên phần lớn thiết bị mạng. Ansible chỉ cần kết nối SSH (giao thức quản trị đã có sẵn trên hầu hết thiết bị mạng), không đòi hỏi cài thêm gì trên thiết bị đích — đây là lý do Ansible trở thành lựa chọn phổ biến hơn hẳn cho riêng mảng tự động hóa thiết bị mạng, trong khi Puppet vẫn giữ vị thế mạnh ở mảng quản lý server (nơi cài agent hoàn toàn khả thi).

## 8. Môi trường cần thiết triển khai Network Automation

Trước khi triển khai bất kỳ công cụ nào ở mục 7, hạ tầng cần đáp ứng 1 số điều kiện tiên quyết:

- **Kết nối quản trị ổn định**: SSH (cho Ansible/Netmiko) hoặc HTTPS/API (cho Catalyst Center) phải được bật và truy cập được từ máy chạy automation tới toàn bộ thiết bị đích.
- **Tài khoản dịch vụ riêng (service account)**: nên dùng 1 tài khoản AAA riêng cho automation (đã học ở [[Bai-27_AAA|Bài 27]]) thay vì dùng chung tài khoản cá nhân — dễ audit, dễ thu hồi quyền nếu cần mà không ảnh hưởng người dùng thật.
- **Quản lý bí mật (credentials) an toàn**: mật khẩu/API key dùng trong script không nên viết thẳng (hardcode) vào file — cần cơ chế lưu trữ riêng (như Ansible Vault, biến môi trường) để tránh lộ khi chia sẻ script.
- **Môi trường thực thi automation**: máy chủ/máy trạm cài sẵn Python, Ansible, thư viện Netmiko và các phụ thuộc liên quan — thường tách riêng khỏi máy trạm cá nhân để dễ kiểm soát.
- **Version Control** (như Git): lưu lịch sử thay đổi của script/Playbook, cho phép rollback và biết chính xác ai đã đổi gì.
- **Môi trường thử nghiệm (lab/staging)**: luôn nên test script/Playbook trên môi trường lab trước khi áp dụng lên hạ tầng thật, tránh rủi ro 1 lỗi logic trong script gây gián đoạn hàng loạt thiết bị cùng lúc — hậu quả của lỗi automation trên diện rộng nghiêm trọng hơn nhiều so với lỗi gõ tay 1 thiết bị.

> [!question] Vì sao 1 lỗi logic nhỏ trong script automation lại có thể nguy hiểm hơn nhiều so với việc gõ nhầm 1 lệnh CLI thủ công trên 1 thiết bị?
> Vì bản chất của automation là **nhân rộng đúng 1 hành động lên nhiều thiết bị cùng lúc** — nếu logic sai (ví dụ script vô tình xóa nhầm VLAN đang dùng thay vì chỉ tạo VLAN mới), lỗi đó sẽ được áp dụng **đồng loạt lên toàn bộ thiết bị nằm trong phạm vi script chạy**, thay vì chỉ ảnh hưởng tới 1 thiết bị như khi gõ nhầm lệnh CLI thủ công trên từng máy. Đây là lý do môi trường thử nghiệm (lab/staging) và cơ chế `--check`/dry-run (sẽ học ở [[Bai-30_Ansible-va-Tong-ket|Bài 30]]) không phải là bước "cho có" mà là yêu cầu bắt buộc trước khi chạy bất kỳ script/Playbook automation nào lên hạ tầng sản xuất thật.

## 9. Verify & Troubleshoot

```
ansible-playbook -i inventory.ini cau-hinh-vlan.yml --check
ansible switches -i inventory.ini -m cisco.ios.ios_command -a "commands='show version'"
```

| Triệu chứng | Nguyên nhân khả dĩ | Cách xử lý |
|---|---|---|
| Gọi REST API tới Controller trả về lỗi 401 | Thiếu hoặc sai thông tin xác thực (token/API key) trong request | Kiểm tra lại cơ chế xác thực API (thường cần lấy token trước qua 1 API endpoint riêng, rồi đính kèm vào các request sau) |
| Gọi REST API trả về lỗi 404 | Sai đường dẫn (endpoint) API, hoặc tài nguyên yêu cầu không tồn tại | Đối chiếu lại tài liệu API (API documentation) của Controller đang dùng, kiểm tra đúng version API |
| Dữ liệu trả về từ API không đọc/parse được | Định dạng dữ liệu trả về (JSON/XML) không đúng như ứng dụng đang mong đợi | Kiểm tra header `Content-Type` của request/response, đối chiếu đúng định dạng đang xử lý (mục 5) |
| Không kết nối được tới thiết bị bằng công cụ automation | Thiếu điều kiện môi trường (SSH chưa bật, service account sai quyền, firewall chặn) | Đối chiếu lại checklist môi trường cần thiết (mục 8) trước khi nghi ngờ lỗi ở script/Playbook |
| Thiết bị mới không tự nhận cấu hình qua ZTP | Sai cấu hình DHCP Option trỏ tới Controller, hoặc thiết bị chưa được khai báo trước trên Controller | Kiểm tra DHCP Option tương ứng; xác nhận đã có "kế hoạch" (plan) cấu hình sẵn cho thiết bị đó trên Controller |

## 10. Lab thực hành

**Môi trường**: 1 công cụ test API (như Postman, hoặc `curl` trên terminal), 1 tài liệu API mẫu (có thể dùng API documentation công khai của Cisco Catalyst Center Sandbox — Cisco cung cấp môi trường sandbox miễn phí để luyện tập).

1. Liệt kê lại checklist môi trường cần thiết (mục 8) và tự đánh giá môi trường lab hiện có của bạn đã đáp ứng những gì, còn thiếu gì.
2. Dùng `curl`/Postman gửi 1 request GET tới 1 API endpoint mẫu, quan sát dữ liệu JSON trả về.
3. So sánh cùng 1 dữ liệu được biểu diễn dưới 3 định dạng JSON/YAML/XML (mục 5), tự tay chuyển đổi qua lại giữa 3 định dạng để làm quen cấu trúc.
4. Vẽ lại sơ đồ kiến trúc SDN 3 lớp (mục 4) cho 1 tình huống cụ thể tự chọn (ví dụ: hệ thống giám sát tự động phát hiện switch bị down và tự động thông báo), xác định rõ đâu là Northbound, đâu là Southbound trong tình huống đó.

**Kết quả kỳ vọng**: làm quen với tư duy gọi API thay vì đăng nhập CLI thủ công, hiểu đúng vai trò từng lớp trong kiến trúc SDN và các công cụ/điều kiện triển khai automation — nền tảng cần thiết trước khi học Python/Netmiko ở [[Bai-30_Ansible-va-Tong-ket|Bài 30]].

## 11. Ngộ nhận thường gặp

| Ngộ nhận | Vì sao sai | Hiểu đúng |
|---|---|---|
| SDN nghĩa là mạng "không cần thiết bị vật lý" nữa | Nhầm lẫn giữa ảo hóa/tập trung hóa Control Plane với việc loại bỏ hạ tầng vật lý | SDN chỉ tách Control Plane ra khỏi thiết bị, Data Plane (chuyển tiếp gói tin thật) vẫn cần switch/router vật lý hoạt động (mục 3) |
| Northbound/Southbound phản ánh sự khác biệt về công nghệ, không chỉ là quy ước đặt tên | Đọc tên gọi theo nghĩa đen | Đây thuần túy là quy ước hình ảnh hóa theo hướng vẽ sơ đồ (trên/dưới), không phản ánh cơ chế kỹ thuật khác biệt ngoài hướng giao tiếp (mục 4) |
| JSON, XML, YAML là 3 ngôn ngữ lập trình khác nhau | Nhầm lẫn định dạng dữ liệu (data format) với ngôn ngữ lập trình | Cả 3 chỉ là cách **biểu diễn dữ liệu có cấu trúc**, không phải ngôn ngữ lập trình — bản thân chúng không có logic xử lý (mục 5) |
| Có Controller/Catalyst Center rồi thì không cần hiểu CLI/cấu hình thủ công nữa | Đánh giá quá cao khả năng thay thế hoàn toàn của automation | Automation dựa trên nền tảng hiểu đúng cách thiết bị hoạt động (đã học xuyên suốt Bài 7-28) — Controller chỉ giúp triển khai nhanh và nhất quán hơn, không thay thế kiến thức nền tảng khi cần troubleshoot sâu |
| Ansible và Puppet là 2 cái tên khác nhau của cùng 1 công cụ | Bỏ qua khác biệt kiến trúc agentless vs agent-based | Ansible agentless (chỉ cần SSH), Puppet thường cần agent cài trên thiết bị đích — khác biệt này quyết định mức độ phù hợp với thiết bị mạng truyền thống (mục 7) |

## 12. Câu hỏi phỏng vấn nền tảng (kèm đáp án chuẩn)

**1. Nêu 3 lợi ích chính của Network Automation so với cấu hình thủ công.**
→ Giảm sai sót do con người, tăng tốc độ triển khai, và đảm bảo tính nhất quán giữa các thiết bị cùng vai trò — ngoài ra còn dễ audit/rollback qua version control và giải phóng nhân lực cho công việc phức tạp hơn.

**2. Phân biệt Data Plane, Control Plane, Management Plane.**
→ Data Plane chuyển tiếp gói tin thực tế qua thiết bị. Control Plane quyết định cách chuyển tiếp (xây dựng bảng định tuyến/MAC, ví dụ qua OSPF/STP). Management Plane cho phép quản trị viên cấu hình/giám sát thiết bị (CLI, SSH, SNMP).

**3. SDN khác Traditional Networking ở điểm cốt lõi nào?**
→ SDN tách riêng Control Plane ra khỏi từng thiết bị, tập trung vào 1 Controller có cái nhìn toàn cục về mạng — cho phép đẩy cấu hình/chính sách xuống toàn bộ thiết bị đồng loạt, thay vì phải cấu hình thủ công từng thiết bị một như mô hình truyền thống.

**4. Phân biệt Northbound API và Southbound API.**
→ Northbound API kết nối Controller với ứng dụng/công cụ tự động hóa phía trên (thường dùng REST/JSON). Southbound API kết nối Controller với thiết bị mạng thực tế phía dưới (thường dùng NETCONF, OpenFlow, SNMP).

**5. Vì sao Ansible phổ biến hơn Puppet cho riêng mảng tự động hóa thiết bị mạng?**
→ Vì Ansible agentless, chỉ cần SSH (đã có sẵn trên hầu hết thiết bị mạng); Puppet thường cần cài agent trên thiết bị đích — điều khó/không khả thi trên phần lớn thiết bị mạng chạy hệ điều hành đóng như Cisco IOS.

## 13. Tài liệu tham khảo

- [Cisco Press — Comparing Traditional Versus Controller-Based Networks](https://www.ciscopress.com/articles/article.asp?p=2995354&seqNum=4)
- [ComputerNetworkingNotes — SDN North-bound and South-bound APIs and Interfaces](https://www.computernetworkingnotes.com/ccna-study-guide/sdn-north-bound-and-south-bound-apis-and-interfaces.html)
- [NetworkersHome — Cisco DNA Center: Network Controller, APIs & Automation](https://www.networkershome.com/fundamentals/network-automation/cisco-dna-center-network-controller/)
- [IPCisco — Cisco DNA Center Overview: 8 Key Features Explained](https://ipcisco.com/lesson/cisco-dna-center/)

---
**Liên kết**: [[Bai-28_Cisco-ASA-Firewall|← Bài 28]] · [[Bai-30_Ansible-va-Tong-ket|Bài 30 - Network Automation với Python →]]
