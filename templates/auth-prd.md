# PRD: Module Auth (Users, Roles, Permissions)

> Phiên bản: 1.1 | Ngày: 09/10/2026 | Trạng thái: Draft
> Tài liệu liên quan: `auth-user-stories.md`, `auth-spec.md`

---

## 1. Bối cảnh & vấn đề

Hệ thống LMS phục vụ 3 nhóm người dùng với nhu cầu và quyền hạn khác nhau (quản trị, giảng viên, học viên). Chưa có cơ chế xác thực và phân quyền, nên chưa thể bảo vệ dữ liệu hay phân tách chức năng. Auth là nền tảng: mọi module sau (courses, lessons, enrollments) đều phụ thuộc vào nó.

## 2. Mục tiêu & chỉ số thành công

| Mục tiêu | Chỉ số đo |
|---|---|
| Người dùng đăng ký và đăng nhập an toàn | 100% mật khẩu được băm bcrypt; 0 trường hợp mật khẩu/token xuất hiện trong response hoặc log |
| Phân quyền đúng theo role | 100% endpoint không công khai đều có guard; có test chứng minh học viên không truy cập được dữ liệu người khác |
| Phiên làm việc mượt | Token hết hạn được gia hạn tự động, người dùng không bị đăng xuất giữa chừng trong 7 ngày hoạt động |
| ADMIN quản lý được người dùng | Tạo, khoá, gán role không cần can thiệp trực tiếp vào database |
| Hiệu năng chấp nhận được | Đăng nhập phản hồi dưới 500 ms (p95) với dữ liệu thử nghiệm |

## 3. Đối tượng người dùng

| Role | Mô tả | Nhu cầu chính với module Auth |
|---|---|---|
| Khách | Chưa có tài khoản | Đăng ký để học |
| STUDENT | Học viên | Đăng nhập, quản lý hồ sơ cá nhân |
| INSTRUCTOR | Giảng viên (do ADMIN tạo) | Đăng nhập, quản lý hồ sơ cá nhân |
| ADMIN | Quản trị viên | Quản lý user, gán role, chỉnh permission |

## 4. Phạm vi

**Trong scope (Phase 1):**
- Đăng ký (chỉ STUDENT tự đăng ký), đăng nhập, đăng xuất, làm mới phiên
- Hồ sơ cá nhân: xem, sửa họ tên, đổi mật khẩu
- ADMIN: tạo, xem, tìm kiếm, khoá/mở khoá user
- Gán role cho user; gán permission cho role
- 3 role cố định: ADMIN, INSTRUCTOR, STUDENT

**Ngoài scope (Phase sau):**
- Quên mật khẩu qua email, xác thực email
- Đăng nhập OAuth (Google, Facebook)
- Xác thực 2 bước (2FA)
- Tạo role mới tuỳ ý qua giao diện
- Nhật ký hoạt động (audit log) chi tiết

## 5. Yêu cầu chức năng

| Mã | Yêu cầu | Ưu tiên | Story |
|---|---|---|---|
| FR-AUTH-01 | Khách tự đăng ký tài khoản STUDENT | Must | US-A01 |
| FR-AUTH-02 | Đăng nhập, đăng xuất, giữ và tự gia hạn phiên làm việc | Must | US-A02, US-A03, US-A04 |
| FR-AUTH-03 | ADMIN tạo tài khoản giảng viên và quản lý user (tìm, lọc, khoá/mở khoá) | Must | US-A06, US-A07, US-A08 |
| FR-AUTH-04 | ADMIN gán role cho user | Must | US-A09 |
| FR-AUTH-05 | Người dùng xem/sửa hồ sơ, đổi mật khẩu | Should | US-A05 |
| FR-AUTH-06 | ADMIN xem và chỉnh permission của role | Could | US-A10 |

Mã yêu cầu khớp với `01-prd.md` (v0.2) của cả dự án.

## 6. Yêu cầu phi chức năng

- **Bảo mật:** bcrypt cho mật khẩu; access token sống 15 phút, refresh token 7 ngày, xoay vòng một lần dùng; refresh token chỉ lưu dạng hash ở server và nằm trong cookie `HttpOnly; Secure; SameSite=Strict` ở trình duyệt (ADR-006)
- **Chống tấn công:** giới hạn 5 lần đăng nhập/phút/IP; thông báo lỗi đăng nhập không tiết lộ email có tồn tại
- **Phân quyền:** kiểm tra ở cả guard và tầng nghiệp vụ
- **Khả năng mở rộng:** permission đặt tên `resource:action` để module sau bổ sung dễ dàng
- **Vận hành:** cấu hình và bí mật đặt trong biến môi trường; thay đổi database qua migration
- **Trải nghiệm:** thông báo lỗi rõ ràng, form validate cả client lẫn server

## 7. Giả định, ràng buộc, rủi ro

**Giả định:**
- Một user có thể có nhiều role
- Permission do dev khai báo trong code, ADMIN chỉ gán/bỏ gán
- Không xoá user thật, chỉ khoá (để giữ dữ liệu học tập)

**Ràng buộc:** tech stack cố định (React, NestJS, SQL Server, JWT).

**Rủi ro:**

| Rủi ro | Giảm thiểu |
|---|---|
| Lộ refresh token | Lưu hash, xoay vòng, phát hiện dùng lại để thu hồi toàn bộ phiên |
| ADMIN tự khoá hoặc gỡ quyền hết quản trị viên | Chặn tự khoá, chặn gỡ role ADMIN cuối cùng |
| Quyền cũ còn hiệu lực sau khi đổi | Permission tra từ cache có hạn ngắn, không nhúng vào token |
| Dò mật khẩu hàng loạt | Rate limit endpoint đăng nhập |

## 8. Câu hỏi còn mở

1. Một user nhiều role hay một role? *(giả định đang dùng: nhiều role)*
2. Phase 1 có giao diện chỉnh permission cho ADMIN không? *(theo PRD dự án: Could, không thuộc nhóm Must của Phase 1)*
3. Có cần xoá user thật không, hay chỉ khoá? *(giả định đang dùng: chỉ khoá)*

## 9. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 09/10/2026 | Bản đầu tiên |
| 1.1 | 09/10/2026 | Đổi mã yêu cầu `FR-01...FR-10` thành `FR-AUTH-01...FR-AUTH-06` (khớp PRD dự án); FR-AUTH-06 là Could; refresh token ở cookie HttpOnly (ADR-006) |
