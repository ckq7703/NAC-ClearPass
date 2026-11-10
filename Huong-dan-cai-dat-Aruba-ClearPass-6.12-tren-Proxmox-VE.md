# Hướng dẫn từng bước: Cài đặt Aruba ClearPass 6.12 trên Proxmox VE 8.3.4

*Biên soạn gốc bởi Kevin Zhu (kevin.zhu@hpe.com) – HPE Aruba Technical Enablement TME. Bản dịch và viết lại tiếng Việt.*

## Giới thiệu

Aruba ClearPass là giải pháp **network access control (NAC)** và quản lý policy mạnh mẽ, giúp tổ chức áp dụng chính sách bảo mật cho các thiết bị truy cập mạng.

> ⚠️ **Lưu ý:** Việc cài đặt ClearPass trên Proxmox VE **không được hỗ trợ chính thức**, hướng dẫn này chỉ dùng cho mục đích lab.

Hướng dẫn này trình bày cách cài **Aruba ClearPass 6.12** trên **Proxmox VE 8.3** bằng **KVM RAW image**, bao gồm:

- Tải image
- Tạo virtual machine (VM) trong Proxmox
- Upload và ghi (write) ClearPass RAW image vào disk
- Cấu hình VM và network
- Khởi động và cấu hình ClearPass lần đầu

Sau khi hoàn tất, bạn sẽ có một instance ClearPass chạy đầy đủ chức năng trên Proxmox VE phục vụ mục đích lab.

---

## Bước 1: Tải ClearPass 6.12 KVM RAW Image

1. **Truy cập HPE Networking Support Portal**
   - Mở trình duyệt, vào `https://networkingsupport.hpe.com/home`.
   - Đăng nhập bằng tài khoản HPE (bắt buộc để truy cập).
2. **Tìm "CPPM"**
   - Gõ `CPPM` vào thanh tìm kiếm, nhấn Enter.
   - Áp dụng filter:
     - **Product:** ClearPass Policy Manager
     - **File Category:** KVM
3. **Tải RAW Image**
   - Tìm **"ClearPass 6.12.0.300732 (Universal KVM RAW Image)"** hoặc phiên bản mới hơn.
   - Nhấn link download và lưu về máy local.

💡 **Mẹo:** File dung lượng lớn, cần đường truyền mạng ổn định.

✅ Giờ bạn đã có image ClearPass 6.12 sẵn sàng để triển khai.

---

## Bước 2: Tạo Virtual Machine trong Proxmox

1. Đăng nhập **Proxmox Web UI**.
2. **Tạo VM mới:** Nhấn **"Create VM"** ở góc trên bên phải.
3. **General Settings:**
   - **Node:** chọn node Proxmox của bạn.
   - **VM ID:** chọn một ID chưa dùng (ví dụ: `224`).
   - **Name:** đặt tên VM (ví dụ: `CPPM-612`).
   - **Start at Boot:** (tùy chọn) tạm thời không tick.
   - Nhấn **Next**.

✅ VM đã được tạo và sẵn sàng để cấu hình.

---

## Bước 3: Cấu hình Virtual Machine Settings

### 1. OS Settings

- **Guest OS Type:** Linux
- **Version:** 6.x — 2.6 Kernel
- **Installation Media:** chọn **Do not use any media**
- Nhấn **Next**.

📌 **Lưu ý:** Cài đặt này đảm bảo ta sẽ tự tay ghi image ClearPass vào disk ở bước sau, thay vì boot từ CD/ISO.

### 2. System Settings

- **Graphics Card:** Default
- **Machine Type:** Default (i440fx)
- **BIOS:** Default (SeaBIOS)
- **SCSI Controller:** VirtIO SCSI single (không dùng tới nhưng giữ mặc định)
- Nhấn **Next**.

✅ Hệ thống đã được cấu hình đúng.

---

## Bước 4: Thêm ổ đĩa ảo (Virtual Disks)

### 1. Disk thứ nhất (System Disk)

- **Bus/Device:** IDE 0
- **Storage:** `local-lvm`
- **Disk Size:** 60GB (tối thiểu)
- **Cache:** Default (No cache)
- Nhấn **Next**.

### 2. Disk thứ hai (Data Disk)

- Nhấn **"Add"** để tạo thêm một disk.
- **Bus/Device:** IDE 1
- **Storage:** `local-lvm`
- **Disk Size:** 40GB (tối thiểu)
- **Cache:** Default (No cache)
- Nhấn **Next**.

✅ VM hiện có 2 disk — một cho hệ thống, một cho dữ liệu.

⚠️ **Mẹo:** Không dùng ổ SCSI hoặc SATA, nếu không bạn sẽ gặp lỗi UUID.

---

## Bước 5: Cấu hình CPU & Memory

### 1. CPU Settings

- **Sockets:** 2
- **Cores per Socket:** 4 (tổng 8 cores)
- **Type:** `x86-64-v2-AES`
- **VCPUs:** 8
- **Enable NUMA:** tick chọn
- Nhấn **Next**.

### 2. Memory Settings

- **Memory (MiB):** `8192` (tối thiểu 8GB, khuyến nghị 12GB)
- Nhấn **Next**.

📌 **Vì sao cần từ 8GB RAM trở lên?**
ClearPass cần nhiều bộ nhớ cho việc xử lý policy và logging. 8GB là mức tối thiểu cho lab; nên nâng lên 12GB nếu có thể.

✅ VM đã được cấu hình CPU và memory.

---

## Bước 6: Cấu hình Network Interfaces

### 1. Network quản lý (cổng đầu tiên)

- **Bridge:** `<bridge của bạn>`
- **Model:** VirtIO (paravirtualized)
- **VLAN Tag:** `<theo môi trường của bạn>`
- **Firewall:** tick chọn
- Nhấn **Next** rồi **Confirm** cấu hình.

### 2. Thêm network dữ liệu (cổng thứ hai)

- Vào **Hardware** → **Add** → **Network Device**.
- **Bridge:** `<bridge của bạn>`
- **Model:** VirtIO (paravirtualized)
- **VLAN Tag:** `<theo môi trường của bạn>` (tham khảo thêm mẹo ở cuối bài về thứ tự interface dựa theo địa chỉ MAC)
- Nhấn **Add**.

📌 **Vì sao cần 2 network interface?**
- **Management Port:** dùng để truy cập quản trị và cấu hình policy.
- **Data Port:** dùng cho xác thực client và lưu lượng RADIUS.

---

## Bước 7: Upload & giải nén ClearPass RAW Image

### 1. Upload file ZIP lên Proxmox

- Mở **WinSCP**, đăng nhập vào server Proxmox.
- Upload file vào thư mục: `/var/lib/vz/template/iso`

### 2. Giải nén Image

SSH vào Proxmox và chạy:

```bash
cd /var/lib/vz/template/iso
unzip CPPM-VM-x86_64-6.12.0.300732-KVM.raw.zip
```

- RAW image sẽ được lưu tại:
  `home/jenkins/KVMImage/CPPM-VM-x86_64-6.12.0.300732-KVM.raw`

⚠️ Đảm bảo image được giải nén đúng trước khi tiếp tục.

✅ Image ClearPass đã được giải nén và sẵn sàng để ghi vào disk.

---

## Bước 8: Ghi RAW Image vào Disk của VM

Sau khi đã upload và giải nén image ClearPass 6.12, ta cần ghi nó vào system disk bằng lệnh `dd`.

### 1. Xác định Logical Volume (LV) của VM

Cần tìm đúng disk để tránh ghi đè nhầm lên VM khác.

- SSH vào Proxmox, chạy:

```bash
lvdisplay
```

- Tìm **logical volume (LV) path** gắn với VM ClearPass của bạn (ví dụ: `vm-224-disk-0` cho VM ID 224).
- Xác định **block device number (dm-ID)** ở trường **Block device**.

Ví dụ output:

```
--- Logical volume ---
LV Path                vm-224-disk-0
LV Name                vm-224-disk-0
VG Name                pve
...
Block device           252:75

--- Logical volume ---
LV Path                vm-224-disk-1
LV Name                vm-224-disk-1
VG Name                pve
...
Block device           252:76
```

Trong ví dụ này, block device là `252:75` cho disk 0, nghĩa là đường dẫn thiết bị là `/dev/dm-75`.

🚨 **Hết sức cẩn thận — ghi nhầm thiết bị sẽ phá hỏng VM khác!**

### 2. Ghi RAW Image vào Disk

Chạy lệnh `dd` để copy image:

```bash
dd if=/var/lib/vz/template/iso/home/jenkins/KVMImage/CPPM-VM-x86_64-6.12.0.300732-KVM.raw of=/dev/dm-<dm-ID của bạn>
```

> Thay `<dm-ID của bạn>` bằng số bạn xác định ở bước trên (ví dụ: `75`).

- Quá trình này sẽ mất một thời gian tùy vào tốc độ disk.

Ví dụ output:

```
94371840+0 records in
94371840+0 records out
48318382080 bytes (48 GB, 45 GiB) copied, 329.859 s, 146 MB/s
```

✅ Image ClearPass đã được ghi thành công vào disk của VM!

---

## Bước 9: Khởi động VM và cấu hình ClearPass

### 1. Khởi động VM

- Trong Proxmox, chọn VM ClearPass.
- Nhấn **"Start"**.
- Mở **Console** để theo dõi quá trình boot.

### 2. Boot Menu của ClearPass

Bạn sẽ thấy boot menu **Aruba ClearPass 6.12**:

```
Aruba ClearPass Platform 6.12.0.300732 [Active]
```

- Để hệ thống tự động boot.

Nếu hệ thống không boot, đảm bảo **IDE 0** được đặt làm boot device đầu tiên:

- Vào **Options → Boot Order**.
- Đảm bảo **IDE0 (ổ hệ thống 60GB)** đứng đầu tiên.

---

## Bước 10: Cài đặt ban đầu (Initial Setup) của ClearPass

### 1. Chọn loại Appliance

Hệ thống sẽ hỏi:

```
Enter appliance type to continue :
1) CLABV
2) C1000V
3) C2000V
4) C3000V
```

- Gõ `1` (**CLABV**) cho mục đích lab, rồi nhấn Enter.

### 2. Bỏ qua cảnh báo cấu hình hệ thống

Hệ thống có thể cảnh báo về RAM và dung lượng disk:

```
WARN: Current active system configuration does not meet the recommended system configuration.
Do you want to continue? [y/N]:
```

Vì chỉ dùng cho lab, cứ bỏ qua cảnh báo này:

- Gõ `y` và nhấn Enter.

### 3. Khởi tạo & mã hóa Disk

Hệ thống sẽ hỏi có muốn mã hóa disk không:

```
Press Y/y to encrypt or any other key to skip encryption: n
```

- Gõ `n` để bỏ qua mã hóa, nhấn Enter.

📌 Mã hóa không cần thiết cho lab và có thể làm tăng thời gian cài đặt.

---

## Bước 11: Cấu hình Networking

Sau khi khởi tạo xong, **System Configuration Wizard** sẽ bắt đầu.

### 1. Đặt Hostname & Management Network

```
Enter hostname: CPPM-DEMO
Enter Management Port IPv4 Address/PrefixLen (Ex:1.1.1.1/24): <IP của bạn>/24
Enter Management Port IPv4 Gateway: <gateway của bạn>
```

⚠️ Nếu gặp lỗi **"Invalid Management Port IPv4"**, nhớ nhập kèm `/24` (subnet mask) khi nhập địa chỉ IP.

### 2. Đặt Primary & Secondary DNS

```
Enter Primary DNS: <DNS server của bạn>
Enter Secondary DNS: <có thể để trống, nhấn Enter>
```

### 3. Bật/Tắt SLAAC cho IPv6

```
Do you want to enable SLAAC mode? [y/n]: n
```

- Gõ `n` và nhấn Enter.

### 4. Đặt mật khẩu Admin

```
New Password: <mật khẩu mới cho user appadmin>
Confirm Password: <nhập lại>
```

---

## Bước 12: Cấu hình cuối cùng (Final Configuration)

### 1. Cấu hình ngày giờ hệ thống (tùy chọn)

```
Do you want to configure system date time information? [y/n]: n
```

- Gõ `n` và nhấn Enter. (Chọn `y` nếu bạn cần cấu hình NTP server, v.v.)

### 2. Bật/Tắt chế độ FIPS

```
Do you want to enable FIPS Mode? [y/n]: n
```

- Gõ `n` và nhấn Enter.

📌 FIPS (Federal Information Processing Standards) Mode không cần thiết cho hầu hết môi trường lab.

### 3. Xem lại cấu hình (Configuration Summary)

Hệ thống sẽ hiển thị bảng tổng kết cấu hình, ví dụ:

```
========================================================
                 Configuration Summary
========================================================
Hostname                              : CPPM-DEMO
Management Port IP Address            : <IP của bạn>
Management Port Subnet Mask           : <mask của bạn>
Management Port Gateway               : <gateway của bạn>
Data Port IP Address                  : <not configured>
Data Port Subnet Mask                 : <not configured>
Data Port Gateway                     : <not configured>
Primary DNS                           : <DNS của bạn>
Secondary DNS                         : <not configured>
SLAAC Mode                            : Disabled
System Date                           : <not configured>
System Time                           : <not configured>
Timezone                              : <not configured>
FIPS Mode                             : False
========================================================
Proceed with the configuration [y[Y]/n[N]/q[Q]]
        y[Y] to continue
        n[N] to start over again
        q[Q] to quit
```

- Nếu mọi thứ đúng, gõ `y` để tiếp tục.

⚠️ Nếu có vấn đề, gõ `n` để quay lại cấu hình.

---

## Bước 13: Hoàn tất cài đặt

### 1. Khởi tạo lần boot đầu tiên

- Hệ thống sẽ format **ổ thứ hai (IDE 1, 40GB)**.
- ClearPass sẽ mất vài phút để khởi tạo.

### 2. Đăng nhập vào ClearPass

Sau khi boot xong, đăng nhập bằng:

```
Username: appadmin
Password: <mật khẩu bạn đã đặt>
```

⚠️ Nếu quên mật khẩu, bạn sẽ cần reset lại từ console của Proxmox.

✅ **Chúc mừng! VM ClearPass 6.12 của bạn đã được cài đặt và cấu hình hoàn chỉnh.**

---

## Bước tiếp theo

- **Truy cập ClearPass Web UI:** mở trình duyệt, vào `https://<management IP>`, áp dụng license Aruba ClearPass Platform.
- Bắt đầu cấu hình network policies.
- Nếu không truy cập được management IP, xem phần mẹo bên dưới.

---

## Mẹo bổ sung: Xác định đúng Management Port trong ClearPass

Trong một số bản cài đặt, management port không phải lúc nào cũng được gán vào `net0` trong Proxmox. Đôi khi nó nằm ở `net1` hoặc interface khác.

📌 Điều này có thể xảy ra do **thứ tự nhận diện interface**: các interface được map dựa theo địa chỉ MAC — **địa chỉ MAC thấp hơn sẽ trở thành management port**, địa chỉ MAC cao hơn trở thành data port.

### Cách xác định Management Port

**1. Đăng nhập ClearPass qua CLI và chạy:**

```bash
show ip
```

- Lệnh này hiển thị địa chỉ IP và MAC của tất cả network interface. Kiểm tra địa chỉ MAC của interface được gắn nhãn **Management Port**.

Ví dụ output:

```
========================================================
Device Type        :  Management Port
--------------------------------------------------------
IPv4 Address       :  <IP của bạn>
Subnet Mask        :  <mask của bạn>
Gateway            :  <gateway của bạn>
IPv6 Address/Prefix Length : <not configured>
Gateway            :  <not configured>
Hardware Address   :  bc:24:11:8a:80:9c
MTU                :  1500
========================================================
Device Type        :  Data Port
--------------------------------------------------------
...
Hardware Address   :  bc:24:11:f0:f0:7d
MTU                :  1500
```

**2. Kiểm tra lại địa chỉ MAC trong Proxmox**

- Vào **Proxmox Web UI**.
- Mở **Hardware → Network Devices**.
- So sánh địa chỉ MAC được liệt kê cho `net0`, `net1`, v.v.

Ví dụ:

```
Network Device (net0)  virtio=BC:24:11:F0:F0:7D, bridge=vmbr1, firewall=1, tag=...
Network Device (net1)  virtio=BC:24:11:8A:80:9C, bridge=vmbr2, firewall=1, tag=...
```

**3. Thiết lập đúng network quản lý**

- Trong ví dụ trên, địa chỉ MAC của management port khớp với `net1` (MAC thấp hơn) thay vì `net0`. Do đó bạn cần cấu hình lại network settings trong Proxmox cho phù hợp — đảm bảo `net1` được kết nối đúng vào bridge dùng cho traffic quản lý.

---

**Chúc mừng, bạn đã hoàn tất cài đặt!**

*Kevin Zhu — kevin.zhu@hpe.com*
*Global Enablement TME – Software & Solutions*
*HPE Aruba Networking*
*CWNE#134 | ACEX#3 ACCX/ACDX/ACMX/ACSX | CCIE*
