# Hướng dẫn cấu hình ClearPass: chỉ cho phép laptop đăng nhập domain và có MAC hợp lệ vào mạng

> **Phiên bản tài liệu:** 1.0 – dùng cho Aruba ClearPass Policy Manager 6.12 (các bản 6.10/6.11 tương tự).
> **Phạm vi:** 802.1X (PEAP-MSCHAPv2) + Active Directory + whitelist địa chỉ MAC + cấu hình switch + cấu hình laptop Windows.
> **Không nằm trong phạm vi:** kiểm tra antivirus (OnGuard), Guest, Onboard.

---

## 0. Lưu ý về nguồn tài liệu (đọc trước)

Khi biên soạn, các trang tài liệu chính thức trên `arubanetworking.hpe.com` (ClearPass User Guide) trả về lỗi **403** và trang `support.hpe.com` không tải được nội dung. Vì vậy:

- Phần **join AD domain, Authentication Source** dựa trên: webinar kỹ thuật chính thức của HPE Aruba (có ảnh chụp màn hình), nội dung trích từ ClearPass 6.12 User Guide qua kết quả tìm kiếm, và các thảo luận trên HPE Airheads.
- Phần **switch** dựa trên tài liệu của Portnox, các thảo luận Airheads/Cisco và tài liệu Aruba Central. **Cú pháp lệnh có thể khác theo model và firmware**, luôn đối chiếu với CLI guide chính thức của dòng switch khách hàng trước khi áp dụng.
- Tên menu trong ClearPass có thể khác nhẹ giữa các phiên bản. Mục nào cần xác nhận lại được đánh dấu **⚠ Cần xác nhận**.
- Danh sách nguồn ở **mục 12** ghi rõ nguồn nào có ảnh minh họa.

---

## 1. Mục tiêu và logic kiểm soát

Chỉ laptop thỏa **cả hai** điều kiện mới được vào mạng:

| # | Đăng nhập tài khoản domain (AD) | MAC nằm trong danh sách cho phép | Kết quả |
|---|---|---|---|
| 1 | Đúng | Có | **Cho phép** |
| 2 | Đúng | Không | **Chặn** |
| 3 | Sai / không đăng nhập | Có | **Chặn** |
| 4 | Sai / không đăng nhập | Không | **Chặn** |

### Giới hạn cần nói rõ với khách hàng

1. PEAP-MSCHAPv2 chỉ chứng minh **"người dùng biết mật khẩu domain"**, **không chứng minh** máy đã join domain. Máy lạ vẫn có thể nhập tài khoản domain hợp lệ, nhưng sẽ bị chặn nếu MAC không có trong whitelist.
2. **MAC có thể bị giả mạo.** Whitelist MAC là lớp bổ sung, không phải lớp bảo mật duy nhất.
3. Một phản hồi trên HPE Airheads (2025) khuyến nghị chuyển sang **EAP-TLS hoặc TEAP** vì MSCHAPv2 có điểm yếu bảo mật. Nếu khách hàng cần mức an toàn cao (chứng minh được máy thuộc domain), nên lên lộ trình **EAP-TLS với certificate cấp từ AD CS** ở giai đoạn sau.

---

## 2. Kiến trúc và luồng xác thực

```
Laptop  --EAPoL-->  Switch access  --RADIUS (UDP 1812/1813)-->  ClearPass  --NTLM/LDAP-->  Domain Controller
(supplicant)        (authenticator)                              (RADIUS server)            (AD)
                          <------------- Access-Accept / Access-Reject -------------
```

1. Laptop cắm vào cổng switch, switch yêu cầu 802.1X.
2. Laptop gửi tài khoản domain (PEAP-MSCHAPv2). Switch chuyển yêu cầu tới ClearPass bằng RADIUS.
3. ClearPass xác thực với AD (NTLM, nên ClearPass **phải join domain**) và tra MAC của laptop trong **Endpoints Repository**.
4. Cả hai đúng: ClearPass trả **Access-Accept** (kèm VLAN/role nếu cấu hình). Một trong hai sai: trả **Access-Reject**.

> Theo webinar của HPE Aruba: ClearPass phải join AD vì khi dùng EAP-PEAP-MSCHAPv2, chỉ có password hash do người dùng gửi lên được dùng để xác thực với AD qua NTLM, và NTLM yêu cầu thiết bị có tư cách thành viên domain.

---

## 3. Thông tin khách hàng cần cung cấp trước khi thực hiện

Gửi bảng này cho khách hàng và thu thập **đầy đủ** trước khi bắt đầu.

### 3.1 Phạm vi và yêu cầu nghiệp vụ

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | Số lượng laptop cần kiểm soát; số người dùng | Xác định license và cỡ whitelist |
| 2 | Mạng dây, Wi-Fi hay cả hai | Wi-Fi cấu hình ở WLC/AP, mạng dây cấu hình ở switch |
| 3 | Vị trí/tòa nhà/tầng cần triển khai đợt đầu (pilot) | Nên làm pilot 1 switch trước |
| 4 | Thiết bị **không phải laptop** cần vào mạng (máy in, camera, IP phone, AP, server) | Cần luồng MAB riêng để không bị chặn nhầm |
| 5 | Máy khách có chạy Windows toàn bộ không (macOS/Linux/điện thoại?) | Hướng dẫn client trong tài liệu này dành cho Windows |
| 6 | Hành vi khi bị chặn: ngắt hẳn (Reject) hay đưa vào VLAN cách ly/guest | Quyết định Enforcement Profile |

### 3.2 ClearPass

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | ClearPass đã cài chưa (VM/appliance), phiên bản | Bản 6.12 theo hướng dẫn cài đặt trước |
| 2 | **Địa chỉ IP/Prefix, gateway, VLAN** của Management Port | Switch phải với tới IP này |
| 3 | Có dùng Data Port không; nếu có thì IP/Prefix/gateway | Mặc định dùng Management Port |
| 4 | Hostname/FQDN dự kiến của ClearPass | Dùng cho certificate |
| 5 | **License** (Platform + Access) đã có chưa | Cần để cấu hình dịch vụ xác thực |
| 6 | IP **DNS server** (phải phân giải được FQDN của domain controller) | Bắt buộc để join domain |
| 7 | IP/FQDN **NTP server** | Lệch giờ quá 5 phút với AD sẽ join/xác thực lỗi |
| 8 | Múi giờ | |
| 9 | Có triển khai cluster/HA (nhiều ClearPass) không | Ảnh hưởng RADIUS server dự phòng và join domain từng node |
| 10 | Ai là người nhận mật khẩu `appadmin`, chính sách đặt mật khẩu | |

### 3.3 Active Directory

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | **Tên domain FQDN** (ví dụ `congty.local`) và **NetBIOS name** (ví dụ `CONGTY`) | |
| 2 | **FQDN và IP của Domain Controller** (chính và dự phòng) | |
| 3 | Mô hình: 1 domain hay nhiều domain/forest | Nhiều forest không có trust thì phải join ClearPass vào từng forest |
| 4 | **Tài khoản có quyền join máy vào domain** (tạm thời, dùng một lần) | Không dùng lâu dài, đổi mật khẩu sau khi xong |
| 5 | **Tài khoản service (bind account)** chỉ đọc để ClearPass truy vấn LDAP, kèm Bind DN | |
| 6 | **Base DN** để tìm user (ví dụ `DC=congty,DC=local`) và OU cụ thể nếu muốn giới hạn | |
| 7 | **Tên AD group** được phép vào mạng (ví dụ `NAC-Allowed-Users`) | Khuyến nghị tạo group riêng để quản lý |
| 8 | Có LDAPS (636) hay LDAP (389) | |
| 9 | Firewall giữa ClearPass và DC: đã mở các cổng cần thiết chưa (xem mục 4) | |
| 10 | Chính sách khóa tài khoản, hết hạn mật khẩu | Ảnh hưởng trải nghiệm khi đăng nhập |

### 3.4 Danh sách MAC hợp lệ (whitelist)

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | File danh sách laptop: **MAC, tên máy, người dùng/phòng ban, ghi chú** | Mẫu CSV ở mục 5.5 |
| 2 | **MAC của card mạng dây và card Wi-Fi** (mỗi card có MAC riêng) | Quên MAC card dock/USB-LAN là lỗi hay gặp |
| 3 | Cơ chế cập nhật khi có máy mới/đổi máy (ai duyệt, ai nhập) | |
| 4 | Có dùng **MAC ngẫu nhiên (Private/Random MAC)** trên Wi-Fi không | Phải tắt trên laptop, nếu không MAC thay đổi liên tục và whitelist không khớp |

### 3.5 Thiết bị mạng

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | **Hãng, model, phiên bản firmware** của từng switch access | Quyết định cú pháp lệnh và khả năng hỗ trợ 802.1X/CoA |
| 2 | Switch có hỗ trợ **802.1X, RADIUS, CoA** không | Một số dòng không hỗ trợ port-access (ví dụ trích từ tài liệu Aruba Central: AOS-CX 8320/8325/8360 không hỗ trợ) |
| 3 | **IP quản trị (hoặc IP nguồn gửi RADIUS)** của từng switch | IP này phải khai báo trong ClearPass Network Devices |
| 4 | Sơ đồ mạng: switch core, access, vị trí ClearPass, cổng uplink | |
| 5 | Danh sách **cổng access** (nối laptop) và **cổng uplink/trunk** | Không bật 802.1X trên cổng uplink |
| 6 | **VLAN** hiện dùng cho laptop; VLAN cách ly/guest (nếu có); DHCP | |
| 7 | **Truy cập quản trị** vào switch (SSH/console) và thời gian bảo trì (change window) | |
| 8 | Có Wi-Fi không: model **WLC/AP**, tên SSID | |
| 9 | Có ACL/firewall giữa switch và ClearPass không | Cần mở UDP 1812, 1813 và UDP 3799 (CoA) |
| 10 | Management VRF có được dùng cho RADIUS không | Ảnh hưởng cú pháp lệnh |

### 3.6 Máy khách Windows

| # | Thông tin | Ghi chú |
|---|---|---|
| 1 | Phiên bản Windows | Windows 10/11 |
| 2 | Có **GPO** để đẩy cấu hình 802.1X không; ai quản trị GPO | Nên đẩy bằng GPO thay vì cấu hình tay |
| 3 | Laptop đã join domain chưa; đăng nhập bằng tài khoản domain hay local | |
| 4 | Có CA nội bộ (AD CS) và đã có certificate cho ClearPass chưa | Cần để client tin cậy RADIUS server (xem mục 5.9) |
| 5 | Chính sách cho phép người dùng tự cắm máy lạ | |

### 3.7 Điều phối triển khai

| # | Thông tin |
|---|---|
| 1 | Người đầu mối: mạng, AD/Windows, bảo mật, helpdesk |
| 2 | Thời gian triển khai pilot và rollout; thời gian bảo trì cho phép |
| 3 | Kế hoạch rollback và người có quyền quyết định |
| 4 | Danh sách người dùng/phòng ban pilot |

---

## 4. Điều kiện tiên quyết (kiểm tra trước khi cấu hình)

### 4.1 Đồng bộ thời gian

Giờ của ClearPass và domain controller **không được lệch quá 5 phút** (theo webinar của HPE Aruba).

- Kiểm tra: **Administration → Server Manager → Server Configuration** → góc trên bên phải có **Set Date & Time**.
- Có thể đặt giờ thủ công hoặc bật **Synchronize time with NTP server** (tối đa 2 NTP server).

### 4.2 DNS

DNS server đặt trên ClearPass phải phân giải được FQDN của domain controller. Lỗi "Server not found in Kerberos database" hoặc "Clock skew too great" khi join thường do DNS hoặc giờ.

### 4.3 Cổng mạng cần mở

| Từ → Đến | Giao thức/cổng | Mục đích |
|---|---|---|
| Switch/AP → ClearPass | UDP **1812** (auth), UDP **1813** (accounting) | RADIUS |
| ClearPass → Switch/AP | UDP **3799** | CoA/Disconnect |
| ClearPass → Domain Controller | DNS (53), Kerberos (88), RPC (135), LDAP (389), SMB (445), LDAPS (636), Global Catalog (3268/3269), và dải RPC động | Join domain, NTLM, LDAP |
| Quản trị viên → ClearPass | TCP **443** (Web UI), TCP **22** (SSH) | |

> **⚠ Cần xác nhận:** HPE không công bố một bảng cổng duy nhất trong các nguồn đã tra. Các thảo luận trên Airheads cho biết lỗi "RPC timeout" khi join domain thường do firewall chặn cổng, và join domain cần mở nhiều cổng hơn so với chỉ LDAP/LDAPS. Nếu có firewall giữa ClearPass và DC, hãy yêu cầu mở rộng rãi trong lúc join rồi thu hẹp sau, hoặc dùng danh sách Microsoft "Service overview and network port requirements for Windows".

---

## 5. Cấu hình trên ClearPass

> Đăng nhập: `https://<IP-management-ClearPass>` (tài khoản quản trị `appadmin`, hoặc tài khoản Policy Manager đã tạo).

### 5.1 License

**Administration → Server Manager → Licensing** → **Add License** và áp dụng license Platform + Access (lab có thể dùng evaluation).

### 5.2 Server Configuration: hostname, DNS, NTP, múi giờ

**Administration → Server Manager → Server Configuration** → chọn node:

- Kiểm tra **hostname**, **DNS** (primary/secondary).
- **Set Date & Time**: bật **Synchronize time with NTP server**, nhập NTP server.

### 5.3 Join ClearPass vào domain AD

Nội dung các bước dưới đây theo webinar HPE Aruba "ClearPass – Dot1x: Purpose of Domain Joining".

1. **Administration → Server Manager → Server Configuration**.
2. Nhấp **tên server ClearPass** muốn join → nhấp **Join AD Domain**.
3. **Domain controller:** nhập **FQDN của domain controller**, nhấn **Tab**. ClearPass hiển thị "Trying to determine the NetBIOS name..." và tự điền **NetBIOS name**.
4. Kiểm tra NetBIOS name đúng. Trong môi trường thật:
   - **Bỏ tick** "Use default domain admin user [Administrator]".
   - Nhập **tài khoản có quyền join máy vào domain** và mật khẩu.
5. Nhấp **Save**. Màn hình hiển thị "Adding host to AD domain", sau đó "**Added host to the domain**". Dịch vụ sẽ khởi động lại.
6. Nhấp **Close**. Trang Server Configuration sẽ liệt kê ClearPass trong mục AD Domains.

> Sau khi join, trong AD xuất hiện một **computer account** cho node ClearPass.

**Chỉ định danh sách DC dùng cho xác thực MSCHAPv2 (tùy chọn):**
**Server Configuration → chọn server → tab System → Modify Password Servers** → nhập tên các domain controller (mỗi dòng một DC) → **Save**. Nếu không khai báo, ClearPass dùng tất cả DC lấy được từ DNS.

**Kiểm tra xác thực với AD bằng CLI** (SSH vào ClearPass, đăng nhập `appadmin`):

```
ad auth -h
```

Dùng lệnh `ad auth` theo hướng dẫn hiển thị để thử một tài khoản domain. Kết quả thành công hiển thị **NT_STATUS_OK: Success (0x0)**.

> **⚠ Cần xác nhận** cú pháp chính xác của `ad auth` theo phiên bản ClearPass bằng `ad auth -h`.

Để rời domain (khi cần làm lại): **Server Configuration → chọn server → Leave AD Domain** → nhập mật khẩu quản trị → **Leave**.

### 5.4 Tạo Authentication Source: Active Directory

**Configuration → Authentication → Sources → Add**:

| Trường | Giá trị |
|---|---|
| **Name** | `AD-Congty` (tùy đặt) |
| **Type** | Active Directory |
| **Use for Authorization** | Bật (để lấy group/thuộc tính) |

Tab **Primary**:

| Trường | Giá trị |
|---|---|
| **Hostname** | FQDN hoặc IP của DC |
| **Port** | 389 (LDAP) hoặc 636 (LDAPS) |
| **Connection Security** | None / LDAPS / StartTLS tùy chính sách |
| **Bind DN** | Tài khoản service chỉ đọc (ví dụ `svc-clearpass@congty.local`) |
| **Bind Password** | Mật khẩu service account |
| **NetBIOS Domain Name** | `CONGTY` |
| **Base DN** | `DC=congty,DC=local` (hoặc OU cụ thể) |
| **Search Scope** | SubTree Search |

Nhấp **Test Connection** để kiểm tra, rồi thêm DC dự phòng ở tab **Backup 1** nếu có.

Tab **Attributes** (lấy nhóm AD):

Theo ClearPass 6.12 User Guide, nếu **không dùng RADIUS** lấy group, dùng filter group `(distinguishedName=%{memberOf})`. Với xác thực RADIUS cần tra **nhóm lồng nhau (nested groups)**, thêm một filter như sau:

| Trường | Giá trị |
|---|---|
| **Filter Name** | `Nested Groups` (ví dụ) |
| **Filter Query** | `(&(objectClass=user)(sAMAccountName=%{Authentication:Username}))` |
| **Attribute Name** | `tokenGroups` |
| **Alias Name** | `Nested Groups` |

> Từ thảo luận Airheads: nếu một người thuộc nhiều AD group, so khớp `memberOf` có thể chỉ trả về một group. Nên dùng thuộc tính nested group ở trên hoặc dùng toán tử **MATCHES_ANY** khi tạo Role Mapping.

> Quan trọng: theo webinar HPE Aruba, với dịch vụ EAP-PEAP-MSCHAPv2, **bước xác thực (kiểm tra mật khẩu) tách biệt với Authentication Source AD**. Nguồn AD này trong trường hợp đó chủ yếu phục vụ **phân quyền (authorization)** (lấy group, thuộc tính).

### 5.5 Khai báo Network Devices (switch / WLC / AP)

**Configuration → Network → Devices → Add**:

| Trường | Giá trị |
|---|---|
| **Name** | `SW-ACCESS-01` |
| **IP or Subnet Address** | **IP nguồn mà switch dùng để gửi RADIUS** (nên cố định bằng source-interface) |
| **Vendor Name** | Aruba / Cisco (đúng hãng) |
| **RADIUS Shared Secret** | Chuỗi bí mật (phải **trùng** với cấu hình trên switch) |
| **Enable RADIUS CoA** | Bật, **RADIUS CoA Port** = `3799` |

Thiếu bước này, ClearPass bỏ qua yêu cầu từ switch (log "unknown NAS" trong Event Viewer).

### 5.6 Tạo whitelist MAC

Mục tiêu: ClearPass biết MAC nào là **hợp lệ**. Có ba cách. **Lưu ý:** Một thảo luận Airheads cho biết **không thể import CSV trực tiếp vào Endpoints Repository** (cần dùng XML hoặc REST API).

**Mẫu CSV cung cấp bởi khách hàng:**

```
mac,hostname,user,department,note
aabbccddeeff,LT-KT-001,nguyenvana,Ke toan,Laptop Dell
112233445566,LT-IT-002,tranvanb,IT,Laptop HP
```

Định dạng MAC thống nhất: 12 ký tự hex, chữ thường, không dấu phân cách (ví dụ `aabbccddeeff`).

#### Cách A: Endpoints Repository (nhập tay, ít máy)

**Configuration → Identity → Endpoints → Add**:

- **MAC Address:** `aabbccddeeff`
- **Description:** tên máy/người dùng
- **Status:** `Known`

#### Cách B: Endpoints Repository qua REST API (nhiều máy)

1. Tạo API client: **Administration → API Services → API Clients → Add** (Operating Mode: ClearPass REST API, Grant Type: client credentials).
2. Lấy token (thay `<cppm-ip>`, `<client_id>`, `<client_secret>`):

```bash
curl -k -X POST https://<cppm-ip>/api/oauth \
  -H "Content-Type: application/json" \
  -d '{"grant_type":"client_credentials","client_id":"<client_id>","client_secret":"<client_secret>"}'
```

3. Tạo endpoint **Known** từ token nhận được (`<token>`):

```bash
curl -k -X POST https://<cppm-ip>/api/endpoint \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"mac_address":"aabbccddeeff","description":"LT-KT-001 nguyenvana","status":"Known"}'
```

4. Import hàng loạt từ file CSV ở trên (bash, bỏ dòng tiêu đề). Dòng nào đã tồn tại sẽ trả lỗi, cần dùng PATCH thay cho POST:

```bash
TOKEN="<token>"
CPPM="<cppm-ip>"
tail -n +2 danhsach-mac.csv | while IFS=, read -r mac host user dept note; do
  curl -k -s -X POST "https://$CPPM/api/endpoint" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"mac_address\":\"$mac\",\"description\":\"$host $user\",\"status\":\"Known\"}"
  echo
done
```

> **⚠ Cần xác nhận:** tên trường JSON và đường dẫn API theo ClearPass Swagger của bản đang dùng: `https://<cppm-ip>/api-docs`. Thử trên 1–2 MAC trước khi chạy hàng loạt.

#### Cách C: ClearPass Guest, Manage Devices (import CSV trực tiếp)

Theo một thảo luận Airheads: đăng ký một thiết bị thủ công ở **Guest → Manage Devices**, **export CSV** để lấy đúng cột, rồi import file của bạn qua **Guest → Manage Devices → More Options**. Các cột gồm `username` (MAC viết hoa, có dấu gạch ngang), `sponsor_name`, `mac_auth` = `1`, `role_id`, `enabled` = `1`, `expire_time` (tùy chọn). Khi dùng cách này, **Authorization Source** của Service phải là **Guest Device Repository** thay vì Endpoints Repository, và điều kiện role mapping cũng khác. Chỉ chọn cách này nếu khách hàng muốn quản lý danh sách thiết bị bằng giao diện Guest.

> Trong phần còn lại của tài liệu dùng **Cách A/B (Endpoints Repository)**.

### 5.7 Tạo Service 802.1X

**Configuration → Services → Add**. Chọn template **802.1X Wired** (mạng dây) hoặc **802.1X Wireless** (Wi-Fi).

**Tab Service:**

| Trường | Giá trị |
|---|---|
| **Name** | `802.1X-Domain-MAC` |
| **Service Rule** | Mặc định của template (`Radius:IETF NAS-Port-Type` ...). Có thể thêm điều kiện theo **Device Group** để chỉ áp dụng cho switch pilot. |

**Tab Authentication:**

| Trường | Giá trị |
|---|---|
| **Authentication Methods** | `[EAP PEAP]` (hoặc bản template mặc định có `[EAP PEAP]`, `[EAP TLS]`, `[EAP MSCHAPv2]`); chỉ giữ phương thức bạn dùng |
| **Authentication Sources** | `AD-Congty` |

**Tab Authorization (rất quan trọng):**

- **Authorization Sources:** thêm **`AD-Congty`** và **`[Endpoints Repository]`**.

> **Cảnh báo (từ thảo luận Airheads):** `[Endpoints Repository]` phải nằm ở **tab Authorization**, **không phải tab Authentication**. Cấu hình sai chỗ làm dịch vụ MAC auth chạy được nhưng 802.1X lỗi.

**Tab Roles → Role Mapping Policy → Add** (tạo trước các role, dùng **Configuration → Identity → Roles**: `Domain-User`, `Allowed-Device`):

| Rule | Điều kiện | Role |
|---|---|---|
| 1 | `Authorization:AD-Congty:memberOf` **MATCHES_ANY** `CN=NAC-Allowed-Users,...` (hoặc `Authentication:Source EQUALS AD-Congty` nếu không dùng group) | `Domain-User` |
| 2 | `Authorization:[Endpoints Repository]:Status` **EQUALS** `Known` | `Allowed-Device` |

Default Role: `[Other]`.

> Theo thảo luận Airheads, thuộc tính `Authorization:[Endpoints Repository]:Status` dùng được trong role mapping (đã được xác nhận với giá trị `Disabled`; `Known` dùng tương tự). Một số người dùng khuyên không lạm dụng `Known` làm điều kiện chung cho nhiều chính sách khác nhau. Trong bài toán này nó chỉ dùng để phân biệt MAC được cấp phép nên phù hợp.

**Tab Enforcement → Enforcement Policy → Add** (tạo trước các Enforcement Profile ở bước tiếp theo):

| Rule | Điều kiện | Enforcement Profile |
|---|---|---|
| 1 | `Tips:Role` **MATCHES_ALL** `Domain-User` `Allowed-Device` | `Allow-Access-Profile` |
| Default | (không khớp) | `[Deny Access Profile]` |

**Enforcement Profile cần tạo** (**Configuration → Enforcement → Profiles → Add**):

- `Allow-Access-Profile`: Template **RADIUS Based Enforcement** (Accept). Nếu muốn gán VLAN, thêm thuộc tính `Tunnel-Type = VLAN (13)`, `Tunnel-Medium-Type = 802 (6)`, `Tunnel-Private-Group-Id = <VLAN-ID>`.
- `[Deny Access Profile]`: có sẵn, trả **Access-Reject**.

> Với yêu cầu chỉ cần **cho vào / không cho vào**, bạn **không cần** gán VLAN; chỉ cần Accept hoặc Reject. Gán VLAN/role phụ thuộc hãng switch (Aruba CX dùng `Aruba-User-Role` hoặc thuộc tính VLAN tùy firmware), nên chỉ làm khi khách hàng yêu cầu.

Nhấp **Save** để hoàn tất Service. Đảm bảo Service **nằm trên các service mặc định** (kéo thả thứ tự ở **Configuration → Services**) để không bị service khác khớp trước.

### 5.8 (Tùy chọn) Service MAB cho thiết bị không có 802.1X

Máy in, camera, v.v. không biết gửi tài khoản domain. Tạo service **MAC Authentication** riêng (**Configuration → Services → Add → template MAC Authentication**), Authorization Source = `[Endpoints Repository]`, cho phép khi `Status EQUALS Known`. Một thảo luận Airheads cho biết thiết bị không thuộc AD sẽ báo "not a member of AD" ở service 802.1X nếu không có service MAB đi kèm.

### 5.9 Certificate cho RADIUS (khuyến nghị)

Khi dùng PEAP, laptop cần **tin cậy certificate** của ClearPass để không hiện cảnh báo hoặc bị lỗi kết nối.

1. **Administration → Certificates → Certificate Store → Server Certificates → Create Certificate Signing Request** (dùng cho `RADIUS/EAP`).
2. Ký CSR bằng CA nội bộ, rồi nhập certificate đã ký (**Import Certificate**).
3. Đẩy **Root CA** vào **Trusted Root Certification Authorities** của laptop bằng GPO.
4. Trong cấu hình PEAP trên laptop, tick **"Verify the server's identity by validating the certificate"** và chọn Root CA đó.

---

## 6. Cấu hình trên switch (access)

Cấu hình **trên switch access**, tức switch có cổng laptop cắm trực tiếp. **Switch core không cần cấu hình 802.1X**, chỉ cần định tuyến VLAN và cho phép lưu lượng tới ClearPass (UDP 1812/1813, UDP 3799).

### 6.1 Nguyên tắc chung

1. Trỏ RADIUS về **IP Management của ClearPass** (nếu Data Port chưa cấu hình), cổng 1812/1813, secret **trùng** với Network Device.
2. Bật 802.1X toàn cục, bật trên **cổng access**, **không** bật trên cổng uplink/trunk/cổng nối ClearPass.
3. Bật **CoA** (dynamic authorization) và khai báo ClearPass là client.
4. IP nguồn RADIUS của switch phải **khớp** với IP khai báo trong ClearPass.
5. Luôn giữ **một cổng quản trị** không bật 802.1X để khỏi khóa mình ra khỏi switch.

### 6.2 Ví dụ Aruba AOS-CX (thay giá trị trong `<...>`)

> **⚠ Cần xác nhận** theo firmware (10.x). Một số bản có thêm tùy chọn như gán server-group theo từng cổng (từ 10.12). Nếu dùng Management VRF thêm `vrf mgmt` vào cuối các lệnh `radius-server host` và `radius dyn-authorization client`.

```
radius-server host <IP-ClearPass> key plaintext <SHARED-SECRET>
aaa group server radius CPPM
    server <IP-ClearPass>

aaa authentication port-access dot1x authenticator
    radius server-group CPPM
    enable

aaa accounting port-access start-stop interim group CPPM

radius dyn-authorization enable
radius dyn-authorization client <IP-ClearPass> secret-key plaintext <SHARED-SECRET>

interface 1/1/10
    aaa authentication port-access dot1x authenticator
        enable
```

Kiểm tra (tham khảo):

```
show aaa authentication port-access interface 1/1/10 client-status
show radius-server
```

Theo một thảo luận Airheads: nếu thấy log "Invalid client in disconnect request", nghĩa là CoA đến từ IP **chưa** được khai báo ở `radius dyn-authorization client`.

### 6.3 Ví dụ Cisco IOS / IOS-XE (Catalyst)

Các lệnh dựa trên mẫu cấu hình trong tài liệu Portnox và hướng dẫn IEEE 802.1X của Cisco. **Không trả lời "Yes" khi lệnh `aaa accounting identity` hỏi chuyển sang CPL** (theo cảnh báo trong tài liệu này, đồng ý có thể không hoàn tác được).

```
aaa new-model
radius server CPPM
 address ipv4 <IP-ClearPass> auth-port 1812 acct-port 1813
 key <SHARED-SECRET>
aaa group server radius CPPM-GRP
 server name CPPM
aaa authentication dot1x default group CPPM-GRP
aaa authorization network default group CPPM-GRP
aaa accounting dot1x default start-stop group CPPM-GRP
dot1x system-auth-control

aaa server radius dynamic-author
 client <IP-ClearPass> server-key <SHARED-SECRET>

ip radius source-interface <Vlan-quan-tri>

interface GigabitEthernet1/0/10
 switchport mode access
 authentication port-control auto
 dot1x pae authenticator
 authentication host-mode multi-auth
```

Kiểm tra:

```
show dot1x all
show authentication sessions interface GigabitEthernet1/0/10
```

> **⚠ Cần xác nhận:** các bản IOS-XE mới dùng chính sách `access-session`/IBNS 2.0 thay cho lệnh `authentication ...` trên cổng. Đối chiếu configuration guide của đúng model và phiên bản.

### 6.4 Wi-Fi (nếu có)

Cấu hình ở WLC/AP: tạo SSID **WPA2/WPA3-Enterprise**, trỏ RADIUS về ClearPass (IP, secret, cổng 1812/1813), khai báo WLC/AP trong ClearPass Network Devices và dùng service **802.1X Wireless**.

---

## 7. Cấu hình laptop Windows

### 7.1 Bật dịch vụ Wired AutoConfig (mạng dây)

Mở PowerShell hoặc CMD **với quyền Administrator**:

```
sc config dot1x start= auto
net start dot1x
```

### 7.2 Cấu hình bằng GPO (khuyến nghị)

Theo tài liệu Microsoft về cấu hình 802.1X cho client có dây bằng Group Policy:

1. Mở **Group Policy Management**, tạo hoặc sửa GPO áp dụng cho OU chứa laptop.
2. **Computer Configuration → Policies → Windows Settings → Security Settings → Wired Network (IEEE 802.3) Policies** (mạng dây) hoặc **Wireless Network (IEEE 802.11) Policies** (Wi-Fi) → **Create A New Policy**.
3. Bật **Use Windows Wired AutoConfig service for clients**.
4. Tab **Security**:
   - **Enable use of IEEE 802.1X authentication**: bật.
   - **Select a network authentication method**: **Microsoft: Protected EAP (PEAP)** → **Properties**.
   - Tick **Verify the server's identity by validating the certificate** và chọn Root CA ở mục 5.9.
   - **Authentication method:** **Secured password (EAP-MSCHAP v2)** → **Configure** → tick **Automatically use my Windows logon name and password (and domain if any)** (theo Microsoft, tùy chọn này chỉ khả dụng khi chọn MS-CHAP v2).
   - **Authentication Mode:** **User or computer authentication** (hoặc **User authentication**).
5. Áp dụng GPO: trên laptop chạy `gpupdate /force` rồi khởi động lại.

### 7.3 Cấu hình tay (chỉ để thử nghiệm)

**Control Panel → Network and Sharing Center → Change adapter settings → chuột phải adapter Ethernet → Properties → tab Authentication**:

- Tick **Enable IEEE 802.1X authentication**.
- Chọn **Microsoft: Protected EAP (PEAP)** → **Settings** → chọn **Secured password (EAP-MSCHAP v2)** → **Configure** → tick **Automatically use my Windows logon name and password (and domain if any)**.

Kiểm tra trạng thái:

```
netsh lan show interfaces
netsh lan show profiles
```

### 7.4 Tắt MAC ngẫu nhiên (Wi-Fi)

**Settings → Network & Internet → Wi-Fi → [SSID] → Random hardware addresses → Off**. Nếu MAC thay đổi liên tục, whitelist sẽ không khớp.

---

## 8. Kế hoạch kiểm thử

Chạy trên **một switch/cổng pilot** trước. Theo dõi tại **Monitoring → Live Monitoring → Access Tracker** (xem từng request, lý do chấp nhận hoặc từ chối).

| # | Kịch bản | Kết quả mong đợi | Kiểm tra |
|---|---|---|---|
| 1 | Laptop có MAC whitelist, đăng nhập tài khoản domain đúng | **Accept** | Access Tracker: Result Accept, Roles gồm `Domain-User` + `Allowed-Device` |
| 2 | Laptop **MAC không** whitelist, tài khoản domain đúng | **Reject** | Roles thiếu `Allowed-Device`, rơi vào Default |
| 3 | Laptop MAC whitelist, **sai mật khẩu** domain | **Reject** | Lỗi xác thực, không có `Domain-User` |
| 4 | Laptop MAC whitelist, tài khoản local (không phải domain) | **Reject** | Lỗi xác thực |
| 5 | Máy lạ MAC lạ, không có tài khoản | **Reject** | |
| 6 | Tài khoản domain bị khóa/hết hạn | **Reject** | |
| 7 | Thiết bị MAB (nếu có) | Theo policy MAB | |
| 8 | Đổi MAC sang `Disabled` hoặc xóa khỏi whitelist rồi cắm lại | **Reject** | |

---

## 9. Xử lý sự cố thường gặp

| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| Access Tracker không thấy request nào | Switch không tới được ClearPass, sai IP/secret, firewall chặn UDP 1812 | `ping`/kiểm tra route; kiểm tra secret; mở UDP 1812/1813 |
| Event Viewer báo **unknown NAS** | IP switch chưa khai báo hoặc khác IP nguồn RADIUS | Khai báo đúng IP trong Network Devices; đặt `ip radius source-interface` |
| Join domain lỗi "Clock skew too great" | Giờ lệch trên 5 phút | Đồng bộ NTP |
| Join domain lỗi "Server not found in Kerberos database" | DNS sai | Trỏ DNS ClearPass về DNS của AD; dùng FQDN |
| Join domain lỗi **RPC timeout** | Firewall chặn cổng giữa ClearPass và DC | Mở các cổng mục 4.3 |
| Lỗi **0xc0000022** khi xác thực MSCHAPv2 | ClearPass chưa join domain hoặc join bị hỏng | Join lại domain (Leave rồi Join) |
| 802.1X lỗi nhưng MAC auth chạy được | `[Endpoints Repository]` đặt nhầm tab Authentication | Chuyển sang tab **Authorization** |
| Tài khoản đúng nhưng vẫn Reject | MAC chưa Known, định dạng MAC khác, MAC ngẫu nhiên | Kiểm tra định dạng `aabbccddeeff`; tắt Random MAC; xem Roles trong Access Tracker |
| Người dùng thuộc nhiều AD group, role gán sai | So khớp `memberOf` chỉ nhận một group | Dùng thuộc tính nested group, hoặc toán tử `MATCHES_ANY` |
| Policy Simulation thành công nhưng máy thật thất bại | Mô phỏng dùng PAP, máy thật dùng MSCHAPv2 | Luôn thử bằng máy thật |
| CoA không hoạt động | Chưa bật dyn-authorization, sai IP client, UDP 3799 bị chặn, chưa bật CoA trong Network Device | Kiểm tra từng điểm; log "Invalid client" cho biết IP không khớp |
| Laptop báo không tin cậy máy chủ | Chưa import Root CA vào laptop | Đẩy Root CA qua GPO |

---

## 10. Kế hoạch triển khai an toàn

1. **Giai đoạn 0, chuẩn bị:** thu thập thông tin ở mục 3, kiểm tra điều kiện ở mục 4.
2. **Giai đoạn 1, ClearPass:** làm mục 5.1 – 5.9. Chưa bật enforcement thật.
3. **Giai đoạn 2, giám sát (monitor):** tạm đặt **Default** của Enforcement Policy là **Allow** (chỉ ghi log), bật 802.1X trên **một cổng pilot**, quan sát Access Tracker để phát hiện MAC thiếu hoặc lỗi cấu hình.
4. **Giai đoạn 3, enforcement:** đổi **Default** thành `[Deny Access Profile]` trên switch pilot, chạy toàn bộ kịch bản mục 8.
5. **Giai đoạn 4, nhân rộng:** thêm từng switch/tầng, chỉnh sửa whitelist theo thực tế.
6. **Rollback:** trên switch tắt 802.1X ở cổng/nhóm cổng bị ảnh hưởng (ví dụ Cisco: `no authentication port-control auto` trên giao diện; AOS-CX: `no enable` trong `aaa authentication port-access dot1x authenticator` của giao diện), hoặc đổi Default Enforcement về Allow.
7. **Sau triển khai:** đổi mật khẩu tài khoản join domain, cập nhật tài liệu, đào tạo helpdesk quy trình thêm MAC mới, lên lộ trình EAP-TLS.

---

## 11. Tóm tắt ngắn

1. Thu thập thông tin khách hàng (mục 3).
2. ClearPass: **NTP + DNS → License → Join AD → Authentication Source AD → Network Device → Whitelist MAC (Known) → Service 802.1X (Authorization: AD + Endpoints Repository) → Role Mapping → Enforcement (Allow nếu đủ 2 role, Default Deny)**.
3. Switch access: **RADIUS + 802.1X + CoA, không bật trên uplink**.
4. Laptop: **GPO PEAP-MSCHAPv2, Wired AutoConfig, import Root CA, tắt Random MAC**.
5. Test bằng Access Tracker, pilot trước, rồi nhân rộng.

---

## 12. Tài liệu tham khảo

> Cột "Hình ảnh" cho biết trang có ảnh chụp màn hình/sơ đồ hay không. Tôi chưa nhúng ảnh vào file này (ảnh thuộc bản quyền của nhà xuất bản); hãy mở link gốc để xem. Mục "Trạng thái" cho biết tôi đã đọc toàn bộ hay chỉ thấy trích đoạn.

| Nguồn | Nội dung dùng | Hình ảnh | Trạng thái |
|---|---|---|---|
| [HPE Aruba Technical Climb Webinar: ClearPass – Dot1x: Purpose of Domain Joining (PDF, 2016)](https://higherlogicdownload.s3.amazonaws.com/HPE/MigratedAssets/Airheads%20Webinar_Clearpass_Domainjoin.pdf) | Join AD, kiểm tra giờ (skew 5 phút), Password Servers, Leave AD Domain, lý do cần join cho PEAP-MSCHAPv2 | Có (ảnh chụp từng bước) | Đã đọc toàn văn (UI 2016, nhưng đường dẫn menu vẫn đúng với bản 6.x) |
| ClearPass Policy Manager 6.12 User Guide: [Configuring the Active Directory Authentication Source](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00007001en_us&page=GUID-ED1E78F0-A6B5-4B39-88C4-477CA5CCF4CC.html&docLocale=en_US) | Tạo AD source, filter group `memberOf`, nested group `tokenGroups` | Có | Chỉ thấy trích đoạn qua tìm kiếm (trang tải về trống) |
| ClearPass 6.12 User Guide: [Preparing for Active Directory Authentication](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00007001en_us&page=GUID-AD94E12E-DFF2-4188-A5C2-5CB05AEBF9A9.html&docLocale=en_US) | Yêu cầu join domain cho 802.1X, lệnh `ad auth` | Có | Chỉ thấy trích đoạn |
| ClearPass 6.12 User Guide: [RADIUS Server Options](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00007001en_us&page=GUID-BAA744D7-2555-4FEB-8ED4-01D822E0BA8F.html&docLocale=en_US) | Tùy chọn tiến trình xác thực AD | Không rõ | Chỉ thấy trích đoạn |
| ClearPass 6.11/6.12 User Guide: [802.1X Wired service template](https://arubanetworking.hpe.com/techdocs/ClearPass/6.11/PolicyManager/Content/CPPM_UserGuide/Services/ServiceTypes8021XWired.htm) | Các tab Service/Authentication/Authorization/Roles/Enforcement; `Endpoints Repository` là nguồn Authorization | Có | **Không tải được (403)**, chỉ thấy trích đoạn |
| [HPE Validated Solution Guide: ClearPass client service configuration](https://arubanetworking.hpe.com/techdocs/VSG/docs/120-policy-deploy/esp-policy-deploy-040-clearpass-client-service-configuration/) | Cấu hình service 802.1X, role mapping, enforcement | Có | **Không tải được (403)**, chỉ thấy trích đoạn; nên mở trực tiếp để xem ảnh |
| Airheads: [ClearPass endpoints known/unknown/disabled](http://community.networking.hpe.com/discussion/clearpass-endpoints-known-unknown-disabled) | `Authorization:[Endpoints Repository]:Status`, đặt repository ở tab Authorization | Không | Đã đọc |
| Airheads: [ClearPass import MACs for MAC auth in bulk](http://community.networking.hpe.com/discussion/clearpass-import-macs-for-macauth-in-bulk) | Import CSV qua Guest → Manage Devices | Không | Đã đọc |
| Airheads: [ClearPass joining AD / lỗi join](https://airheads.hpe.com/discussion/clearpass-join-ad-domain) và [cổng ClearPass–AD](https://community.arubanetworks.com/discussion/what-are-the-right-ports-for-cppm-to-ad) | Xử lý sự cố join domain, cổng | Không | Chỉ thấy trích đoạn |
| Airheads: [AOS-CX 10.12 per-port RADIUS server group](https://airheads.hpe.com/discussion/aos-cx-1012-security-update-per-port-radius-server-group) và [CoA invalid client address](https://airheads.hpe.com/discussion/aruba-cx-coa-invalid-client-address) | Lệnh port-access, dyn-authorization trên AOS-CX | Không | Chỉ thấy trích đoạn |
| [Portnox: Aruba 802.1X switch configuration](https://doc.portnox.com/topics/nas_switches_aruba) | Cấu trúc lệnh RADIUS và port-access trên Aruba | Không | Đã đọc (trộn cú pháp AOS-Switch và AOS-CX, đã loại phần không dùng) |
| [Portnox: Cisco 802.1X switch configuration](https://doc.portnox.com/topics/nas_switches_cisco) | Cấu trúc lệnh Cisco IOS/IOS-XE | Không | Đã đọc |
| [Aruba Central: Configuring Authentication on AOS-CX](https://arubanetworking.hpe.com/techdocs/central/latest/content/nms/aos-cx/cfg/conf-cx-auth.htm) | Các dòng switch hỗ trợ port-access | Có | **Không tải được (403)**, chỉ thấy trích đoạn |
| [Microsoft Learn: EAP for network access in Windows](https://learn.microsoft.com/windows-server/networking/technologies/extensible-authentication-protocol/network-access) | Ý nghĩa các tùy chọn PEAP/MSCHAPv2, "Automatically use my Windows logon name and password" | Không | Đã đọc (một phần) |
| Microsoft: [Configure 802.1X Wired Access Clients for PEAP-MS-CHAP v2 Authentication](https://technet.microsoft.com/en-us/library/dd759154.aspx) | GPO Wired Network (IEEE 802.3) Policies | Có | Chỉ thấy trích đoạn (tài liệu Windows Server 2008 R2, giao diện hiện tại có thể khác) |
| Cisco: [Cisco ISE Secure Wired Access Prescriptive Deployment Guide](https://community.cisco.com/t5/security-documents/ise-secure-wired-access-prescriptive-deployment-guide/m-p/3641515) | Tham khảo cấu hình 802.1X trên Catalyst 9300 (dùng ISE, chỉ để so sánh cú pháp) | Có | Chỉ thấy trích đoạn |

### Việc cần làm để hoàn thiện tài liệu này

1. Mở trực tiếp các trang **HPE (403/trống)** bằng trình duyệt có đăng nhập, đối chiếu tên trường và menu với phiên bản ClearPass thực tế.
2. Đối chiếu **cú pháp switch** với CLI guide đúng model và firmware của khách hàng.
3. Chạy thử toàn bộ trên **lab hoặc một switch pilot** trước khi triển khai thật.
