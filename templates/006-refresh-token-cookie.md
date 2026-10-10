# ADR-006: Lưu refresh token ở cookie HttpOnly

- **Trạng thái:** Đã chấp nhận (09/10/2026); `auth-spec.md` đã cập nhật v1.1 theo ADR này
- **Ngày:** 09/10/2026

## Bối cảnh
`auth-spec.md` hiện thiết kế refresh token nằm trong body (client tự lưu). Với SPA, nếu lưu token ở `localStorage`, một lỗ hổng XSS có thể đánh cắp được cả refresh token (sống 7 ngày). Hệ thống có hiển thị nội dung do người dùng nhập (Markdown bài học, ghi chú, nhận xét), nên XSS là rủi ro đáng cân nhắc.

## Quyết định (đề xuất)
- **Access token**: giữ trong bộ nhớ của app (biến JS), không ghi vào storage
- **Refresh token**: cookie `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`, JS không đọc được
- `POST /auth/login` trả access token trong body và set cookie; `POST /auth/refresh` đọc cookie (không còn nhận `refreshToken` trong body); `POST /auth/logout` thu hồi và xoá cookie
- Khi tải lại trang, frontend gọi `POST /auth/refresh` để khôi phục phiên

## So sánh
| Phương án | Ưu | Nhược |
|---|---|---|
| Cả hai token ở `localStorage` | Đơn giản nhất | XSS lấy được cả hai; rủi ro cao nhất |
| **Access trong bộ nhớ + refresh ở cookie HttpOnly** | XSS không đọc được refresh token; CSRF được giảm nhiều nhờ SameSite và cookie chỉ gửi tới `/auth` | Phức tạp hơn một chút; cần cấu hình domain, CORS credentials |
| Cả hai ở cookie | Không cần quản lý token ở JS | Mọi endpoint chịu rủi ro CSRF, cần cơ chế chống CSRF toàn bộ |

## Điều kiện
- Frontend và API cùng **site** (cùng domain đăng ký, ví dụ `app.example.com` và `api.example.com`) để `SameSite=Strict` hoạt động; môi trường dev dùng proxy của dev server
- CORS bật `credentials` và chỉ cho đúng domain frontend
- Kiểm tra thêm header `Origin` ở `/auth/refresh` và `/auth/logout`

## Hệ quả
- `auth-spec.md` (v1.1) đã cập nhật: endpoint refresh/logout bỏ `refreshToken` khỏi body, bổ sung cookie và kiểm tra Origin
- Frontend: lớp HTTP gọi refresh **một lần tại một thời điểm** (single-flight) và đồng bộ giữa các tab để tránh hai tab cùng dùng một refresh token (token xoay vòng một lần dùng; dùng lại sẽ bị coi là bị đánh cắp và thu hồi toàn bộ phiên)
- Ghi chú: Xem lại cơ chế "đợi ngắn" (grace window) ở phía server nếu thấy người dùng bị đăng xuất oan khi mở nhiều tab

## Xem lại khi
Frontend và API buộc phải khác site (không dùng chung domain đăng ký).
