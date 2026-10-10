# User Stories: Module Auth (Users, Roles, Permissions)

> Phiên bản: 1.1 | Ngày: 09/10/2026
> Nguồn yêu cầu: `auth-prd.md` | Thiết kế kỹ thuật: `auth-spec.md`

## Định nghĩa "Xong" (Definition of Done)

- Đáp ứng toàn bộ Acceptance Criteria (AC) của story
- Có DTO validate, trả lỗi đúng status code
- Có test cho các story đánh dấu (T)
- Không lộ mật khẩu, token, stack trace ra client
- Chạy được theo hướng dẫn kiểm tra và đã commit theo Conventional Commits

---

## Epic 1: Xác thực

### US-A01: Đăng ký tài khoản (FR-AUTH-01, Must)
Với vai trò **khách**, tôi muốn **đăng ký tài khoản** để **học các khoá học**.

AC:
- Nhập email, mật khẩu, họ tên; email chưa tồn tại
- Mật khẩu tối thiểu 8 ký tự, có chữ hoa, chữ thường và số
- Tài khoản tạo ra luôn có role STUDENT, bỏ qua mọi trường role gửi lên
- Email trùng (không phân biệt hoa/thường) thì báo lỗi rõ ràng

### US-A02: Đăng nhập (FR-AUTH-02, Must) (T)
Với vai trò **người dùng đã có tài khoản**, tôi muốn **đăng nhập** để **sử dụng hệ thống**.

AC:
- Đúng email và mật khẩu thì nhận access token (trong phản hồi) và refresh token (trong cookie HttpOnly)
- Sai email hoặc sai mật khẩu đều trả cùng một thông báo chung
- Tài khoản bị khoá thì không đăng nhập được
- Quá 5 lần/phút thì bị chặn tạm thời

### US-A03: Tự động gia hạn phiên (FR-AUTH-02, Must) (T)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **phiên tự gia hạn** để **không bị đăng xuất giữa chừng**.

AC:
- Access token hết hạn thì tự lấy access token mới bằng refresh token trong cookie; refresh token cũng được xoay vòng
- Mỗi refresh token chỉ dùng được một lần
- Dùng lại token đã dùng thì thu hồi toàn bộ phiên của user đó
- Refresh token hết hạn hoặc đã thu hồi thì yêu cầu đăng nhập lại
- Tải lại trang thì phiên được khôi phục nếu refresh token còn hiệu lực

### US-A04: Đăng xuất (FR-AUTH-02, Must)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **đăng xuất** để **kết thúc phiên an toàn**.

AC:
- Refresh token hiện tại bị thu hồi ngay và cookie bị xoá
- Dùng lại token đó để refresh sẽ bị từ chối

## Epic 2: Hồ sơ cá nhân

### US-A05: Quản lý hồ sơ và mật khẩu (FR-AUTH-05, Should)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **xem, sửa hồ sơ và đổi mật khẩu** để **giữ thông tin tài khoản chính xác và an toàn**.

AC:
- Xem được thông tin cá nhân (không có mật khẩu)
- Chỉ sửa được họ tên; không tự đổi email, role, status
- Đổi mật khẩu phải nhập đúng mật khẩu cũ
- Đổi mật khẩu xong, mọi refresh token cũ bị thu hồi

## Epic 3: Quản trị người dùng (ADMIN)

### US-A06: Xem danh sách user (FR-AUTH-03, Must)
Với vai trò **ADMIN**, tôi muốn **xem danh sách user có tìm kiếm, lọc, phân trang** để **quản lý hiệu quả**.

AC:
- Lọc theo role, status; tìm theo email hoặc họ tên
- Phân trang `page`, `limit` (mặc định 1 và 20, tối đa 100)
- Người không phải ADMIN nhận 403

### US-A07: Tạo tài khoản giảng viên (FR-AUTH-03, Must)
Với vai trò **ADMIN**, tôi muốn **tạo tài khoản cho giảng viên** để **họ bắt đầu tạo khoá học**.

AC:
- Nhập email, họ tên, mật khẩu tạm, danh sách role
- Email không được trùng

### US-A08: Khoá/mở khoá tài khoản (FR-AUTH-03, Must) (T)
Với vai trò **ADMIN**, tôi muốn **khoá hoặc mở khoá tài khoản** để **xử lý vi phạm**.

AC:
- Khoá xong, user không đăng nhập được và mọi refresh token bị thu hồi
- Access token còn sống cũng bị từ chối ở request kế tiếp
- ADMIN không tự khoá chính mình

## Epic 4: Phân quyền

### US-A09: Gán role cho user (FR-AUTH-04, Must) (T)
Với vai trò **ADMIN**, tôi muốn **gán role cho user** để **cấp quyền phù hợp**.

AC:
- Thay toàn bộ danh sách role của user bằng danh sách mới
- Không được gỡ role ADMIN của quản trị viên cuối cùng

### US-A10: Chỉnh permission của role (FR-AUTH-06, **Could**)
Với vai trò **ADMIN**, tôi muốn **xem và chỉnh permission của từng role** để **kiểm soát chức năng**.

AC:
- Xem được danh sách permission và permission của mỗi role
- Thay danh sách permission của role; hiệu lực sau tối đa 1 phút hoặc ngay khi cache bị xoá
- Không được gỡ permission quản trị (`roles:manage`, `users:update`) khỏi role ADMIN

---

## Thứ tự thực hiện đề xuất

1. US-A01, US-A02, US-A03, US-A04 (nền xác thực)
2. US-A05
3. US-A06, US-A07, US-A08
4. US-A09 (US-A10 là Could, làm khi còn thời gian)

## Truy vết (Traceability)

| Story | Yêu cầu PRD | Phần Spec |
|---|---|---|
| US-A01 | FR-AUTH-01 | API Auth, bảng `users`, `user_roles` |
| US-A02 | FR-AUTH-02 | API Auth, luồng đăng nhập |
| US-A03 | FR-AUTH-02 | Bảng `refresh_tokens`, luồng refresh |
| US-A04 | FR-AUTH-02 | API Auth |
| US-A05 | FR-AUTH-05 | API Users (`/users/me`) |
| US-A06 | FR-AUTH-03 | API Users |
| US-A07 | FR-AUTH-03 | API Users |
| US-A08 | FR-AUTH-03 | API Users, edge case khoá user |
| US-A09 | FR-AUTH-04 | API Users (`PUT /users/:id/roles`) |
| US-A10 | FR-AUTH-06 | API Roles & Permissions |

## Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 09/10/2026 | Bản đầu tiên |
| 1.1 | 09/10/2026 | Đổi mã yêu cầu sang `FR-AUTH-xx` (khớp PRD dự án và `02-user-stories.md`); US-A10 là Could; AC phiên làm việc cập nhật theo ADR-006 (refresh token ở cookie HttpOnly) |
