# Hướng dẫn ClearPass 6.14: Authentication Source AD, Network Device, Endpoint, Service 802.1X Wired và Enforcement Policy

> **Nguồn:** *ClearPass Policy Manager 6.14 Online Help* (HPE, file PDF `HPE_sd00007218en_us`), đọc trực tiếp các mục liệt kê ở mục 9.
> **Điều kiện trước:** ClearPass đã join domain AD (xem [Huong-dan-tich-hop-AD-vao-ClearPass.md](Huong-dan-tich-hop-AD-vao-ClearPass.md)).
> **Mục tiêu cuối:** laptop cắm vào switch chỉ vào mạng khi **tài khoản domain đúng** và **MAC nằm trong whitelist**.
> **Chưa có ảnh:** hướng dẫn này chưa kèm ảnh chụp màn hình. Ảnh trong PDF chưa được trích.

**Cách đọc ghi chú nguồn.** Mỗi mục ghi rõ phần nào lấy từ tài liệu 6.14 và phần nào là thiết kế đề xuất, cần thử trên lab. Phần đề xuất được đánh dấu **[Đề xuất, cần thử]**.

---

## 1. Thứ tự cấu hình

| Bước | Việc cần làm | Menu trong ClearPass |
|---|---|---|
| 1 | Join domain AD | Administration > Server Manager > Server Configuration |
| 2 | Tạo Authentication Source AD | Configuration > Authentication > Sources |
| 3 | Khai báo switch (Network Device) | Configuration > Network > Devices |
| 4 | Nhập MAC vào whitelist (Endpoint) | Configuration > Identity > Endpoints |
| 5 | Tạo Enforcement Policy (và Role Mapping) | Configuration > Enforcement > Policies |
| 6 | Tạo Service 802.1X Wired | Configuration > Services |
| 7 | Cấu hình switch và laptop | Ngoài ClearPass |

Tạo Authentication Source, Network Device và Endpoint trước, vì Service và Enforcement Policy sẽ tham chiếu tới chúng.

---

## 2. Tạo Authentication Source loại Active Directory

### 2.1 Vai trò của Authentication Source (theo tài liệu 6.14)

- Với **EAP-PEAP-MSCHAPv2**, việc kiểm tra mật khẩu do cơ chế **join domain** đảm nhiệm. Authentication Source AD ở đây **chỉ dùng cho phân quyền** (authorization): ClearPass tra user trong AD bằng LDAP, lấy các thuộc tính như nhóm (group) để dùng trong role mapping và enforcement.
- Với EAP-TLS, không cần join domain.
- Authentication Source được ClearPass **bind** vào AD bằng một tài khoản có quyền tìm kiếm user. Tài khoản này cần ít nhất quyền **domain user**.

### 2.2 Các bước

1. Vào **Configuration > Authentication > Sources**, bấm **Add**.
2. Tab **General**:

| Trường | Giá trị |
|---|---|
| **Name** | Tên nguồn, ví dụ `Corp-AD`. Tên này dùng khi tạo role mapping và enforcement policy. |
| **Description** | Mô tả, tùy chọn. |
| **Type** | `Active Directory`. |
| **Use for Authorization** | Bật (mặc định). Cho phép ClearPass lấy thuộc tính để role mapping. |
| **Authorization Sources** | Nguồn bổ sung để lấy thuộc tính role mapping. Tùy chọn. |
| **Server Timeout** | Số giây chờ trước khi coi server không phản hồi. |
| **Cache Timeout** | Số giây cache thuộc tính đã lấy. |
| **Backup Servers Priority** | Thứ tự chuyển sang server dự phòng. Hãng **khuyến nghị cấu hình ít nhất một backup server** (nút **Add Backup**). |

3. Bấm **Next** sang tab **Primary**:

| Trường | Ý nghĩa và giá trị |
|---|---|
| **Hostname** | Tên hoặc IP của máy chủ AD. Máy này phải là LDAP server (hầu hết domain controller đều là LDAP server). |
| **Connection Security** | Chọn **AD over SSL** để dùng cổng **636**. Hãng cảnh báo rằng nếu không dùng SSL thì tên và mật khẩu bind đi qua mạng dạng **clear text**. Khi dùng SSL, phải thêm **CA certificate của AD** vào **Certificate Trust List** của ClearPass. |
| **Port** | LDAP: 389. LDAP over SSL: 636. Với forest nhiều domain dùng Global Catalog: 3268 (không SSL), 3269 (có SSL). |
| **Verify Server Certificate** | Bật để kiểm tra certificate của server khi kết nối bảo mật. |
| **Bind DN** | DN đầy đủ của tài khoản có quyền tìm user, ví dụ `CN=svc-clearpass,CN=Users,DC=corp,DC=local`. Với domain đơn, tài khoản này phải nằm cùng nhánh và bên dưới Base DN. |
| **Bind Password** | Mật khẩu của tài khoản bind. |
| **NetBIOS Domain Name** | ClearPass tự điền. |
| **Base DN** | DN điểm bắt đầu tìm kiếm. Bắt buộc với domain đơn. Hãng **khuyến nghị thu hẹp Base DN** tới OU hoặc container chứa user (không dùng giá trị tự điền) để giảm tải cho AD. Bấm **Search Base DN** để duyệt cây LDAP, đây cũng là cách thử kết nối tới AD. Với forest nhiều domain thì để trống Base DN. |
| **Search Scope** | `Subtree Search` (mọi đối tượng và đối tượng con), `One-Level Search` (ngay dưới Base DN), hoặc `Base Object`. |
| **LDAP Referrals** | Hãng khuyến nghị **không bật** *Follow Referrals* vì có thể gây vòng lặp tìm kiếm. |
| **Bind User** | *Allow bind using user password* (mặc định bật). Với TACACS+, ClearPass luôn bind bằng chính user. |
| **User Certificate** | Giữ giá trị mặc định, trừ khi quản trị AD dùng thuộc tính khác. |
| **Always use NetBIOS name** | Bật nếu muốn dùng giá trị trong NetBIOS Domain Name để xác thực, thay vì domain trong thuộc tính User-Name của RADIUS. |

4. Bấm **Next** sang tab **Attributes**. Tab này định nghĩa các **filter LDAP** và thuộc tính lấy về. Giữ các filter mặc định cho lần đầu, xem mục 2.3.
5. Bấm **Save**. Danh sách Authentication Sources hiện nguồn mới với thông báo *Authentication source ... added*.

### 2.3 Các filter mặc định đáng chú ý

Theo tài liệu 6.14, filter mặc định của Active Directory gồm:

| Filter | Mục đích |
|---|---|
| **Authentication** | Tìm user theo tên đăng nhập, lấy các thuộc tính như `memberOf`, `mail`, `displayName`, `accountExpires`. |
| **Group** | `(distinguishedName=%{memberOf})`. Lấy tên nhóm (`cn`) mà user hoặc máy thuộc về. |
| **Machine** | `(&(objectClass=computer)(sAMAccountName=%{Host:Name}$))`. Tìm computer account trong AD. `%{Host:Name}` là tên máy kết nối nếu có. Các thuộc tính lấy về gồm `dNSHostName`, `operatingSystem`, `operatingSystemServicePack`. |

**Filter Machine** là cơ sở để kiểm tra "máy có trong domain" (xem mục 8).

### 2.4 Nhóm lồng nhau (nested group)

Nếu user thuộc nhóm lồng nhiều cấp, tài liệu hướng dẫn tạo thêm filter dùng thuộc tính `tokenGroups`:

- **Filter Query:** `(&(objectClass=user)(sAMAccountName=%{Authentication:Username}))`
- **Attribute Name:** `tokenGroups`, **Alias Name:** `Nested Groups`, **Data Type:** `String`.
- Dùng **LDAP browser** (tab Browse) để chọn nhóm và sao chép giá trị `objectSid`. Giá trị này đặt vào rule của enforcement policy.
- Mục này chỉ áp dụng cho xác thực RADIUS.

---

## 3. Khai báo switch (Network Device)

Switch phải có trong danh sách thiết bị của ClearPass thì ClearPass mới xử lý yêu cầu RADIUS từ nó.

1. Vào **Configuration > Network > Devices**, bấm **Add**.
2. Tab **Device**:

| Trường | Ý nghĩa |
|---|---|
| **Name** | Tên thiết bị. |
| **IP Address or Subnet** | IPv4 hoặc IPv6, hoặc subnet. Có thể dùng khoảng, ví dụ `192.168.1.1-20`. |
| **Description** | Mô tả. |
| **RADIUS Shared Secret** | Mật khẩu chung giữa switch và ClearPass. Phải **trùng** với cấu hình trên switch. |
| **TACACS+ Shared Secret** | Chỉ dùng khi quản trị switch bằng TACACS+. |
| **Vendor Name** | Hãng thiết bị (ví dụ Aruba, Cisco). ClearPass nạp từ điển RADIUS tương ứng. |
| **Enable RADIUS Dynamic Authorization** | Cho phép ClearPass gửi CoA hoặc disconnect tới thiết bị. Cổng mặc định **3799**, thiết bị phải nhận được cổng này từ ClearPass. |
| **Enable RadSec** | Chỉ bật nếu dùng RadSec (TCP 2083). Với 802.1X thông thường thì không bật. |

3. Bấm **Add** để lưu.

Lưu ý của hãng về **subnet**: khi khai báo cả subnet, các thiết bị trong subnet chỉ được đọc thông tin SNMP nếu ClearPass nhận được trap từ chúng.

---

## 4. Whitelist MAC (Endpoint)

### 4.1 Nhập tay một MAC

1. Vào **Configuration > Identity > Endpoints**, bấm **Add**.
2. Điền:

| Trường | Ý nghĩa |
|---|---|
| **MAC Address** | Địa chỉ MAC của laptop. Khi tìm kiếm, ClearPass hiểu các dạng `112233AABBCC`, `11:22:33:aa:bb:cc`, `11-22-33-AA-BB-CC`, `1111-2222-3333`, `1111.2222.3333`. |
| **Description** | Mô tả (khuyến nghị), ví dụ tên người dùng hoặc số tài sản. |
| **Status** | `Known client`, `Unknown client` hoặc `Disabled client`. `Known client` và `Unknown client` dùng được trong role mapping qua thuộc tính `Authentication:MacAuth`. `Disabled client` dùng để chặn một MAC cụ thể. |
| **Attributes** | Thuộc tính tùy chỉnh (kiểu String). Mọi thuộc tính đã nhập đều có thể dùng trong Rules Editor của role mapping. |

3. Bấm **Save**.

### 4.2 Nhiều MAC

Tài liệu 6.14 liệt kê các chức năng *Performing Bulk Updates of Endpoint Attributes* (cập nhật hàng loạt thuộc tính) cùng nhập và xuất qua XML hoặc API. Chi tiết import hàng loạt cần đọc thêm trong mục đó nếu số máy lớn.

### 4.3 Đề xuất thiết kế whitelist **[Đề xuất, cần thử]**

- Đặt **Status = Known client** cho các MAC hợp lệ, và thêm thuộc tính tùy chỉnh, ví dụ `Allowed = true`, để Enforcement Policy kiểm tra rõ ràng.
- Chọn thuộc tính tùy chỉnh thay vì chỉ dựa vào Status giúp tách "MAC đã được cấp phép" khỏi "MAC ClearPass tự học khi thiết bị đã từng kết nối".
- Cần thử trên lab xem **Endpoints Repository** có tự trả thuộc tính của MAC khi xác thực PEAP hay không, và cách tham chiếu thuộc tính đó trong rule (thường là qua `Endpoint:<tên thuộc tính>` hoặc Authorization Source `Endpoints Repository`).

---

## 5. Role Mapping (tùy chọn, nên dùng)

Role Mapping gán **role** cho yêu cầu dựa trên thuộc tính lấy từ AD hoặc Endpoint. Enforcement Policy sau đó dùng role để quyết định quyền truy cập.

- Tạo ở **Configuration > Identity > Role Mappings**, hoặc ngay khi tạo Service (tab **Roles**).
- Mỗi policy có **Default Role** (role mặc định khi không rule nào khớp) và **Rules Evaluation Algorithm** (hiển thị rule khớp đầu tiên).
- Ví dụ rule đề xuất **[Đề xuất, cần thử]**:
  - Nếu `Corp-AD:memberOf` chứa nhóm `Staff` thì role `Staff-User`.
  - Nếu `Endpoint:Allowed` bằng `true` thì role `Allowed-Device`.

---

## 6. Enforcement Policy

### 6.1 Các thông số (theo tài liệu 6.14)

1. Vào **Configuration > Enforcement > Policies**, bấm **Add**.
2. Tab **Enforcement**:

| Trường | Giá trị |
|---|---|
| **Name**, **Description** | Tên và mô tả. |
| **Enforcement Type** | Chọn **RADIUS** (cho 802.1X). Các loại khác: TACACS+, WebAuth (SNMP/CLI)/CoA, Application, Event. |
| **Default Profile** | Profile áp dụng khi **không rule nào khớp**. Đặt profile **từ chối truy cập** (hoặc VLAN cách ly) để mặc định là chặn. |

3. Tab **Rules**: bấm **Add Rule**, mở **Rules Editor**:
   - Điều kiện (Conditions) có thể dùng thuộc tính thuộc các namespace **Tips:Role**, **Tips:Posture** và **Date**. Giá trị của `Tips:Role` có thể là role định nghĩa trong ClearPass hoặc role lấy từ nguồn phân quyền.
   - Mỗi rule gán một hoặc nhiều **Enforcement Profile**. Khi rule khớp, thuộc tính của profile được gửi tới thiết bị truy cập mạng.
   - Dùng **Move Up/Move Down** để đổi thứ tự rule. Rule đầu tiên khớp sẽ được áp dụng.
4. Mỗi service chỉ gắn **một** enforcement policy.

### 6.2 Thiết kế rule cho yêu cầu "domain đúng và MAC hợp lệ" **[Đề xuất, cần thử]**

| Thứ tự | Điều kiện | Profile |
|---|---|---|
| 1 | Role chứa `Staff-User` **và** role chứa `Allowed-Device` | Cho phép (ví dụ `[Allow Access Profile]` hoặc gán VLAN công ty) |
| Mặc định | Không khớp | Từ chối (`[Deny Access Profile]`) hoặc VLAN cách ly |

Các tên profile mặc định cần kiểm tra lại trên giao diện (mục **Configuration > Enforcement > Profiles**) trước khi dùng.

---

## 7. Service 802.1X Wired

Theo tài liệu 6.14, mục *802.1X Wired Service*:

1. Vào **Configuration > Services**, bấm **Add**.
2. Tab **Service**:

| Trường | Giá trị |
|---|---|
| **Type** | `Aruba 802.1X Wired`. Mẫu này dùng cho client nối qua mạng LAN Ethernet có xác thực 802.1X. Tài liệu không nói rõ mẫu này có giới hạn theo hãng switch hay không, nên cần thử trên lab nếu dùng switch hãng khác. Có mẫu `802.1X Wired - Identity Only` nếu không cần posture và audit. |
| **Name** | Tên service. |
| **Monitor Mode** | Chỉ giám sát, không áp dụng chế độ chặn. Hữu ích khi thử nghiệm ban đầu, **nhưng nhớ tắt khi triển khai chính thức**. |
| **More Options** | Hiện thêm các tab: Authorization, Posture Compliance, Audit End-hosts, Profile Endpoints, Accounting Proxy. Với yêu cầu này, chỉ cần **Authorization**. |
| **Service Rule** | Điều kiện để một yêu cầu RADIUS kích hoạt service này. Mẫu đã có sẵn rule, có thể sửa. |

3. Bấm **Next** sang tab **Authentication**:

| Trường | Giá trị |
|---|---|
| **Authentication Methods** | Mặc định có `EAP PEAP`, `EAP FAST`, `EAP TLS with CN check`, `EAP TTLS`, `EAP MSCHAPV2`. Thứ tự **quan trọng**: ClearPass đề xuất phương thức đầu tiên. Đặt `EAP PEAP` lên đầu nếu phần lớn client dùng PEAP. Có thể xóa các phương thức không dùng. |
| **Authentication Sources** | Chọn nguồn xác thực. Với yêu cầu này, thêm `Corp-AD` (mục 2). Tối đa 23 nguồn. |
| **Strip Username Rules** | Bật nếu cần bỏ tiền tố hoặc hậu tố tên đăng nhập (ví dụ `CORP\`). |
| **Service Certificate** | Chọn certificate nếu cần. |

4. Tab **Authorization** (hiện sau khi chọn **More Options > Authorization**): ClearPass lấy thuộc tính role mapping từ nguồn gắn với authentication source và nguồn gắn với service. Thêm nguồn bổ sung (ví dụ `Endpoints Repository`) vào mục *Additional authorization sources from which to fetch role-mapping attributes* để lấy thuộc tính của MAC. **[Đề xuất, cần thử]**
5. Tab **Roles**: chọn **Role Mapping Policy** (mục 5).
6. Tab **Enforcement**: chọn **Enforcement Policy** (mục 6). Trường này **bắt buộc**. Tùy chọn **Use Cached Results** dùng lại role và posture từ phiên trước.
7. Tab **Summary**: kiểm tra rồi **Save**.

---

## 8. Về yêu cầu "laptop đã join domain"

Tài liệu 6.14 và phần trước cho thấy:

- PEAP-MSCHAPv2 kiểm tra **tài khoản và mật khẩu** người dùng. Nó không tự chứng minh laptop đã join domain.
- ClearPass có sẵn filter **Machine** `(&(objectClass=computer)(sAMAccountName=%{Host:Name}$))` để tra computer account trong AD, và tùy chọn **Machine Authentication Cache Timeout** (mặc định 24 giờ) trong Service Parameters > Policy Server.
- Trong log của tài liệu có ví dụ username dạng `host/Asif-Test-PC2`. Đây là cách Windows gửi tài khoản **máy** khi xác thực máy (machine authentication).

Hướng thiết kế **[Đề xuất, cần thử]**:
1. Cấu hình GPO để laptop xác thực bằng **computer account** (`host/<tên máy>`), hoặc xác thực cả máy và người dùng.
2. Enforcement Policy thêm điều kiện dựa trên kết quả tra filter Machine (máy có tồn tại trong AD).
3. Hoặc chuyển sang **EAP-TLS** với certificate máy cấp từ AD CS. Khi đó không cần join domain cho ClearPass, và certificate chứng minh máy thuộc tổ chức. Đây là hướng chắc chắn nhất.

Phần 6 của tài liệu 6.14 mà tôi tìm thấy mang tên *Configuring 802.1X Authentication with Machine Authentication* nhưng nói về **ArubaOS Mobility Access Switch**, không phải ClearPass, nên không dùng cho phần này.

---

## 9. Các mục đã đọc trong tài liệu 6.14

| Nội dung | Mục trong PDF |
|---|---|
| Join AD, NTP, password servers | *Joining a Policy Manager Server to an Active Directory Domain* |
| Authentication Source AD | *Adding Active Directory as an Authentication Source to Policy Manager* |
| Filter mặc định AD | Mục tham số của *Generic LDAP and Active Directory* (phần mô tả filter) |
| Nested group | *Configuring Deep-Nested Active Directory Queries* |
| Network Device | *Adding and Modifying Network Devices > Adding a Network Device* |
| Endpoint | *Adding and Modifying Endpoints > Manually Adding an Endpoint* |
| Role Mapping | *Configuring a Role and Role-Mapping Policy* (chỉ đọc lướt) |
| Enforcement Policy | *Configuring Enforcement Policies > Adding an Enforcement Policy* |
| Service 802.1X Wired | *802.1X Wired Service* |

## 10. Việc còn lại

1. Đọc mục **Role Mapping** kỹ hơn và kiểm tra cách tham chiếu thuộc tính Endpoint trong rule.
2. Đọc mục **Enforcement Profiles** để lấy tên profile cho phép và từ chối chính xác.
3. Cấu hình switch (cần biết hãng và model).
4. Cấu hình GPO trên laptop, kể cả xác thực máy.
5. Thử trên lab và bổ sung ảnh.
