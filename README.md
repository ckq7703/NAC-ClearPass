# NAC-ClearPass

Tài liệu triển khai **Aruba ClearPass** (Network Access Control) bằng tiếng Việt.

## Nội dung

| File | Mô tả |
|---|---|
| [Huong-dan-cai-dat-Aruba-ClearPass-6.12-tren-Proxmox-VE.md](Huong-dan-cai-dat-Aruba-ClearPass-6.12-tren-Proxmox-VE.md) | Cài đặt ClearPass 6.12 trên Proxmox VE 8.3.4 bằng KVM RAW image (chỉ dùng cho lab, không được hãng hỗ trợ chính thức). |
| [Huong-dan-ClearPass-802.1X-Domain-va-MAC-hop-le.md](Huong-dan-ClearPass-802.1X-Domain-va-MAC-hop-le.md) | Cấu hình 802.1X (PEAP-MSCHAPv2) + Active Directory + whitelist MAC: chỉ laptop đăng nhập domain và có MAC hợp lệ mới vào được mạng. Gồm checklist thông tin khách hàng, cấu hình ClearPass, switch, laptop Windows, kiểm thử và xử lý sự cố. |
| [Installing Aruba ClearPass 6.12 on Proxmox VE.pdf](<Installing Aruba ClearPass 6.12 on Proxmox VE.pdf>) | Tài liệu gốc (tiếng Anh) của Kevin Zhu, HPE Aruba, làm nguồn cho hướng dẫn cài đặt. |

## Lưu ý

- Cú pháp lệnh trên switch (Aruba CX, Cisco) cần đối chiếu với CLI guide đúng model và firmware trước khi áp dụng.
- Các mục đánh dấu **⚠ Cần xác nhận** trong tài liệu là phần chưa kiểm chứng được với tài liệu chính thức.
- Nên thử trên lab hoặc một switch pilot trước khi triển khai thật.
- Tài liệu PDF gốc thuộc bản quyền của tác giả và HPE; chỉ lưu để tham khảo nội bộ.
