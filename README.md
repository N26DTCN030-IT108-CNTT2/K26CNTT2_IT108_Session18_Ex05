# Báo cáo: Kiến trúc Phân phối Secret An toàn (Zero-Knowledge)

## 1. Thiết kế Kiến trúc Phân phối (Data Flow)
Mô hình Zero-Knowledge đảm bảo nền tảng lưu trữ code (GitHub) và các kênh giao tiếp (Chat, Email) hoàn toàn không biết giá trị thực của Secret.

**Quy trình triển khai:**
1. **Khởi tạo bộ khung:** Quản trị viên (Admin) tạo tệp `.env.example` định nghĩa tên biến và thiết lập `.gitignore` để chặn tệp `.env`, sau đó đẩy (push) lên GitHub.
2. **Lưu trữ tập trung (Vault):** Admin tạo Secret thật (AWS Key, OpenAI Key) và lưu trữ vào một dịch vụ Secret Manager chuyên dụng có mã hóa đầu cuối (ví dụ: Bitwarden Secrets Manager, HashiCorp Vault, hoặc Doppler). 
3. **Phân quyền truy cập:** Admin cấp quyền **Read-only** (Chỉ đọc) trên Secret Manager cho 4 thành viên còn lại. Không ai được phép gửi key qua nhóm chat.
4. **Đồng bộ về máy trạm (Local):** 
   - Dev tải mã nguồn từ GitHub về máy (`git clone`).
   - Dev nhân bản file cấu trúc: `cp .env.example .env`.
   - Dev đăng nhập vào Secret Manager bằng tài khoản cá nhân để copy các giá trị Key thật và dán vào file `.env` trên máy cục bộ.

## 2. Xử lý rủi ro bảo mật (Edge Case): Thành viên rời nhóm
Khi một thành viên rời đi, ta không thể xóa file `.env` vật lý trên máy tính cá nhân của họ. Rủi ro lộ lọt dữ liệu là cực kỳ cao. Do đó, hệ thống phải thực hiện quy trình **Xoay vòng khóa (Secret Rotation)** ở tầng Server.

**Quy trình vô hiệu hóa và cập nhật:**
1. **Thu hồi quyền truy cập (Revoke Access):** Quản trị viên lập tức xóa tài khoản của cựu thành viên khỏi GitHub Org, Cloud Workspace và hệ thống Secret Manager.
2. **Hủy và tạo khóa mới (Rotate Keys):** 
   - Đăng nhập vào AWS IAM Console: Xóa (Delete) hoặc Vô hiệu hóa (Deactivate) Access Key cũ. Tạo Access Key mới.
   - Đăng nhập vào OpenAI Platform: Thu hồi (Revoke) API Key cũ và tạo API Key mới.
3. **Cập nhật hệ thống trung tâm:** Admin thay thế các giá trị khóa mới lên Secret Manager.
4. **Cập nhật máy trạm:** Thông báo cho các thành viên còn lại vào Secret Manager để lấy khóa mới cập nhật vào file `.env` của họ. 
*(Hệ quả: File `.env` trên máy của cựu thành viên vẫn tồn tại nhưng chứa các khóa đã bị server từ chối cấp phép - HTTP 401 Unauthorized, đảm bảo an toàn tuyệt đối cho hệ thống).*
