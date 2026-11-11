# NAC-ClearPass

Tài liệu triển khai **Aruba ClearPass** (Network Access Control) bằng tiếng Việt.

Mục tiêu của bộ tài liệu: laptop cắm vào switch chỉ được vào mạng khi **đăng nhập tài khoản domain đúng** và **MAC nằm trong whitelist**. Có chặn theo giờ (21:00 đến 04:00 sáng) và danh sách thiết bị miễn trừ.

## Nội dung

| File | Mô tả |
|---|---|
| [Huong-dan-ClearPass-802.1X-Domain-va-MAC-hop-le.md](Huong-dan-ClearPass-802.1X-Domain-va-MAC-hop-le.md) | **Hướng dẫn chính.** Triển khai từng giai đoạn theo thứ tự: thông tin khách hàng cần cung cấp (có ví dụ điền), chuẩn bị, nền tảng ClearPass (join AD, certificate), Authentication Source, whitelist MAC, Network Device, Role Mapping, Enforcement Policy, Service 802.1X, cấu hình laptop (GPO), cấu hình switch **Dell N-Series và Cisco SG350**, pilot, kiểm thử, xử lý sự cố. Có chính sách chặn theo giờ và thiết bị miễn trừ qua MAB. Kèm ảnh chụp màn hình từ ClearPass 6.14. |
| [Huong-dan-tich-hop-AD-vao-ClearPass.md](Huong-dan-tich-hop-AD-vao-ClearPass.md) | Join ClearPass vào Active Directory: NTP, múi giờ, join và leave domain, Password Servers. Có ảnh minh họa từng bước. |
| [Huong-dan-tich-hop-AD-vao-ClearPass-co-anh.md](Huong-dan-tich-hop-AD-vao-ClearPass-co-anh.md) | Bản của file trên có **ảnh nhúng sẵn** (base64), mở ở đâu cũng hiện ảnh. Nội dung giống nhau. Nếu sửa nội dung thì sửa file gốc rồi tạo lại bản này. |
| [Huong-dan-ClearPass-6.14-Source-Service-Enforcement.md](Huong-dan-ClearPass-6.14-Source-Service-Enforcement.md) | Tham chiếu tham số theo tài liệu ClearPass 6.14: Authentication Source AD, Network Device, Endpoint, Service 802.1X Wired, Enforcement Policy. Dùng để tra cứu từng trường. |
| [Huong-dan-cai-dat-Aruba-ClearPass-6.12-tren-Proxmox-VE.md](Huong-dan-cai-dat-Aruba-ClearPass-6.12-tren-Proxmox-VE.md) | Cài đặt ClearPass 6.12 trên Proxmox VE 8.3.4 bằng KVM RAW image (chỉ dùng cho lab, không được hãng hỗ trợ chính thức). |
| [Installing Aruba ClearPass 6.12 on Proxmox VE.pdf](<Installing Aruba ClearPass 6.12 on Proxmox VE.pdf>) | Tài liệu gốc (tiếng Anh) của Kevin Zhu, HPE Aruba, làm nguồn cho hướng dẫn cài đặt. |
| [HPE_sd00007218en_us_ClearPass Policy Manager 6.14 Online Help.pdf](<HPE_sd00007218en_us_ClearPass Policy Manager 6.14 Online Help.pdf>) | Tài liệu gốc HPE: ClearPass 6.14 Online Help (bản mới nhất, nguồn chính của các hướng dẫn). |
| [HPE_sd00007000en_us_ClearPass Policy Manager 6.11 Online Help.pdf](<HPE_sd00007000en_us_ClearPass Policy Manager 6.11 Online Help.pdf>) | Tài liệu gốc HPE: ClearPass 6.11 Online Help (dùng để so sánh, nguồn ảnh join AD). |
| [HPE_sd00007984en_us_HPE Networking ClearPass 6.14 Installation Guide.pdf](<HPE_sd00007984en_us_HPE Networking ClearPass 6.14 Installation Guide.pdf>) | Tài liệu gốc HPE: ClearPass 6.14 Installation Guide. |
| [images/](images/) | Ảnh minh họa dùng trong các hướng dẫn: `ad-join/` (join AD, ClearPass 6.11), `clearpass-614/` (ClearPass 6.14), `switch-sg350/` (Cisco SG350). |

## Thứ tự đọc gợi ý

1. Đọc **mục 1 đến 4** của hướng dẫn chính: mục tiêu, kiến trúc, thông tin khách hàng, lộ trình.
2. Gửi **mục 3** cho khách hàng điền.
3. Làm theo **mục 5 đến 13** đúng thứ tự giai đoạn. Không đảo thứ tự, vì giai đoạn sau dùng kết quả của giai đoạn trước.
4. Cần tra từng tham số của ClearPass thì mở file 6.14 và file join AD.

## Lưu ý

- Cú pháp switch Dell N-Series và Cisco SG350 đã đối chiếu với CLI guide của hãng, nhưng vẫn cần kiểm tra đúng model và firmware. Đặc biệt **Dell N3224T-ON** và **Cisco SG350** không nằm hoàn toàn trong tài liệu đã đọc.
- Ví dụ switch Aruba CX và Cisco Catalyst (phụ lục A của hướng dẫn chính) chưa đối chiếu với tài liệu hãng.
- Các mục đánh dấu **⚠ Cần xác nhận** hoặc **[Đề xuất, cần thử]** là phần chưa kiểm chứng với tài liệu chính thức, cần thử trên lab.
- Nên thử trên lab hoặc một switch pilot trước khi triển khai thật.
- Ảnh và tài liệu gốc thuộc bản quyền của HPE, Cisco, Dell và tác giả. Chỉ dùng làm tài liệu nội bộ.
- Danh sách thiết bị `SW-Wireless.xlsx` **không đưa lên repo**, chỉ giữ ở máy.
