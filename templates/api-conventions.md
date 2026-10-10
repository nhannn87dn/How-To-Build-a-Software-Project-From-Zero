# Quy ước API chung

> Phiên bản: 0.1 (Draft) | Ngày: 09/10/2026
> Mọi module tuân theo tài liệu này. Request/response chi tiết từng endpoint nằm trong spec module.
> Liên quan: `architecture.md`, `erd.md`

---

## 1. Quy ước chung

| Hạng mục | Quy ước |
|---|---|
| Base URL | `/api/v1` (đổi phá vỡ tương thích thì lên `/api/v2`) |
| Định dạng | JSON, UTF-8; JSON dùng `camelCase` |
| Xác thực | Header `Authorization: Bearer <accessToken>`; refresh token ở cookie HttpOnly (ADR-006) |
| Đặt tên tài nguyên | Danh từ số nhiều, chữ thường, gạch ngang: `/courses`, `/lesson-progress` |
| Quan hệ cha-con | **Lồng** khi con không tồn tại độc lập: `/courses/:courseId/chapters`. **Phẳng** khi con có định danh riêng và hay truy cập trực tiếp: `/lessons/:id`, `/submissions/:id` |
| Hành động chuyển trạng thái | `PATCH /<resource>/:id/status` kèm `{ status }`; service kiểm tra chuyển đổi hợp lệ |
| Thay cả tập hợp | `PUT` (ví dụ `PUT /users/:id/roles`) |
| Giá trị enum | `UPPER_SNAKE_CASE`: `DRAFT`, `PUBLISHED`, `ENDED`, `PENDING`, `APPROVED`, `REJECTED`; trạng thái bài nộp dẫn xuất: `NOT_SUBMITTED`, `SUBMITTED`, `GRADED`, `OVERDUE` |
| ID | Số nguyên; trong JSON là số |

### HTTP method và status code

| Method | Dùng cho | Thành công |
|---|---|---|
| GET | Đọc | 200 |
| POST | Tạo mới | 201 (trả đối tượng vừa tạo) |
| PUT | Thay thế hoàn toàn / upsert idempotent | 200 |
| PATCH | Sửa một phần | 200 |
| DELETE | Xoá | 204 (không body) |

| Mã lỗi HTTP | Khi nào |
|---|---|
| 400 | Dữ liệu không hợp lệ (validate) |
| 401 | Chưa xác thực / token hết hạn hoặc sai |
| 403 | Đã xác thực nhưng không đủ quyền (sai vai trò, không phải chủ khoá) |
| 404 | Không tồn tại, **hoặc** đối tượng riêng tư của người khác (xem `architecture.md` mục 7.2) |
| 409 | Xung đột trạng thái hoặc trùng (đăng ký trùng, hết chỗ, chuyển trạng thái không hợp lệ, quá hạn nộp) |
| 429 | Quá giới hạn tần suất (kèm header `Retry-After`) |
| 500 | Lỗi hệ thống (không lộ chi tiết) |

## 2. Cấu trúc response

**Thành công:**
```json
{ "success": true, "data": { }, "message": "OK" }
```

**Danh sách (có phân trang):**
```json
{
  "success": true,
  "data": { "items": [], "page": 1, "limit": 20, "total": 134 },
  "message": "OK"
}
```

**Lỗi:**
```json
{
  "success": false,
  "message": "Bạn đã đăng ký khoá này",
  "code": "ENROLLMENT_DUPLICATE",
  "errors": [ { "field": "email", "message": "Email không hợp lệ" } ]
}
```
- `message`: tiếng Việt, hiển thị được cho người dùng
- `code`: mã nội bộ **ổn định**, frontend dùng để xử lý logic và ánh xạ thông báo; không đổi tên tuỳ tiện
- `errors`: chỉ có khi `VALIDATION_FAILED`, mỗi phần tử gắn với một trường
- Không bao giờ trả stack trace hay thông tin nội bộ

**Ngoại lệ:** endpoint tải tệp (xuất CSV) trả tệp thật (`Content-Type: text/csv; charset=utf-8`, `Content-Disposition: attachment; filename="bang-diem-<ten-khoa>-<dd-mm-yyyy>.csv"`); lỗi của nó vẫn theo cấu trúc JSON ở trên.

## 3. Phân trang, lọc, sắp xếp

| Tham số | Quy tắc |
|---|---|
| `page` | Từ 1, mặc định 1 |
| `limit` | Mặc định 20, tối đa 100; vượt thì cắt về 100 |
| `search` | Tìm chứa chuỗi (không phân biệt hoa/thường) trên các trường được phép của endpoint |
| Lọc | Tham số riêng theo tên trường: `?status=PENDING&role=STUDENT` |
| `sort` | `field:asc|desc`, ví dụ `sort=createdAt:desc`; **chỉ chấp nhận các trường nằm trong danh sách cho phép** của endpoint (chống sắp xếp theo cột tuỳ ý) |

Mọi endpoint trả danh sách có thể tăng trưởng đều **bắt buộc phân trang** (kể cả danh sách bài nộp, học viên).

## 4. Thời gian

- Truyền nhận: ISO-8601 UTC, ví dụ `"2026-10-09T03:00:00.000Z"`.
- Ngày thuần (ngày bắt đầu/kết thúc khoá): `"YYYY-MM-DD"`.
- Frontend chuyển sang **GMT+7** khi hiển thị, định dạng `dd/mm/yyyy HH:mm`.
- **Hạn nộp bài và mọi kiểm tra theo thời gian đều dùng giờ server**, không tin giờ client.

## 5. Giới hạn dữ liệu mặc định (DTO)

| Trường | Giới hạn |
|---|---|
| `email` | Tối đa 255 ký tự, đúng định dạng, chuẩn hoá chữ thường |
| `password` | 8-72 ký tự (giới hạn của bcrypt), có chữ hoa, chữ thường, số |
| `fullName`, tiêu đề (khoá, chương, bài học, bài tập) | 1-100 / 1-200 ký tự |
| Mô tả khoá, đề bài | Tối đa 5.000 ký tự |
| Nội dung bài học (Markdown) | Tối đa 50.000 ký tự |
| `note` (ghi chú nộp bài), `feedback` | Tối đa 2.000 / 5.000 ký tự |
| `linkUrl`, `videoUrl` | Tối đa 500 ký tự, bắt buộc `https://`; `videoUrl` chỉ nhận host YouTube/Google Drive |
| Kích thước body JSON | Tối đa 1 MB |
| `score` | 0 đến `maxScore`, tối đa 1 chữ số thập phân |

## 6. Bảo mật ở tầng API

- `ValidationPipe`: `whitelist`, `forbidNonWhitelisted`, `transform`.
- Không bao giờ nhận `role`, `status`, `instructorId`, `studentId` từ body ở những chỗ chúng phải suy ra từ token hoặc từ quy trình (ví dụ `studentId` của bài nộp lấy từ token).
- Rate limit: `/auth/login` 5 lần/phút/IP; toàn cục khoảng 100 yêu cầu/phút/IP.
- Header `X-Request-Id` trong mỗi response, có trong log để truy vết. Server tự sinh UUID; chỉ giữ lại id do client gửi khi nó khớp `^[A-Za-z0-9-]{8,64}$` (chống chèn nội dung vào log).
- Lỗi mức framework (`HttpException` thường, không phải `AppException`) luôn trả thông báo tiếng Việt cố định theo mã HTTP; lỗi nghiệp vụ dùng `AppException` với `code` và `message` tường minh. Lỗi 5xx không bao giờ trả chi tiết gốc.
- CORS: chỉ domain frontend, bật credentials (do dùng cookie).

## 7. Mã lỗi nội bộ (danh mục khởi điểm)

| Mã | HTTP | Ý nghĩa |
|---|---|---|
| `VALIDATION_FAILED` | 400 | Dữ liệu sai định dạng; kèm `errors[]` |
| `AUTH_INVALID_CREDENTIALS` | 401 | Sai email hoặc mật khẩu (thông báo chung) |
| `AUTH_TOKEN_INVALID` | 401 | Access token sai/hết hạn |
| `AUTH_REFRESH_INVALID` | 401 | Refresh token sai/hết hạn/đã thu hồi |
| `AUTH_ACCOUNT_LOCKED` | 403 | Tài khoản bị khoá |
| `FORBIDDEN` | 403 | Không đủ quyền |
| `NOT_FOUND` | 404 | Không tồn tại hoặc không có quyền biết |
| `USER_EMAIL_TAKEN` | 409 | Email đã tồn tại |
| `USER_LAST_ADMIN` | 409 | Không được gỡ/khoá quản trị viên cuối cùng |
| `USER_CANNOT_LOCK_SELF` | 409 | ADMIN tự khoá chính mình |
| `ROLE_PROTECTED_PERMISSION` | 409 | Không được gỡ permission quản trị khỏi role ADMIN |
| `AUTH_CURRENT_PASSWORD_WRONG` | 400 | Mật khẩu hiện tại không đúng khi đổi mật khẩu |
| `COURSE_NOT_PUBLISHED` | 409 | Khoá chưa công bố |
| `COURSE_ENDED` | 409 | Khoá đã kết thúc |
| `COURSE_NO_LESSONS` | 409 | Công bố khoá chưa có bài học |
| `COURSE_HAS_STUDENTS` | 409 | Không thể đưa về Nháp vì đã có học viên |
| `COURSE_FULL` | 409 | Đủ sĩ số tối đa |
| `ENROLLMENT_DUPLICATE` | 409 | Đã có đăng ký cho khoá này |
| `ENROLLMENT_NOT_APPROVED` | 403 | Chưa được duyệt vào khoá |
| `CHAPTER_HAS_PROGRESS` | 409 | Chương có dữ liệu học tập, không xoá được |
| `LESSON_IN_USE` | 409 | Bài học đã có tiến độ hoặc bài tập gắn vào |
| `ASSIGNMENT_HAS_SUBMISSIONS` | 409 | Bài tập đã có bài nộp |
| `SUBMISSION_PAST_DUE` | 409 | Đã quá hạn nộp |
| `SUBMISSION_ALREADY_GRADED` | 409 | Bài đã chấm, không nộp lại được |
| `SCORE_OUT_OF_RANGE` | 400 | Điểm ngoài thang |
| `RATE_LIMITED` | 429 | Quá giới hạn tần suất |
| `BAD_REQUEST` | 400 | Yêu cầu không hợp lệ nhưng không thuộc dạng validate (lỗi mức framework) |
| `SERVICE_UNAVAILABLE` | 503 | Dịch vụ phụ thuộc chưa sẵn sàng (ví dụ database) |
| `INTERNAL_ERROR` | 500 | Lỗi hệ thống |

Mỗi module thêm mã mới vào bảng này khi viết spec; mã đã công bố không đổi tên.

## 8. Bản đồ endpoint (tổng quan)

Chi tiết body, response, lỗi từng endpoint: viết ở spec module. Cột "Quyền" ghi permission và điều kiện sở hữu theo `architecture.md` mục 7.

### Auth, Users, Roles (E1)

| Method | Endpoint | Quyền | Story |
|---|---|---|---|
| POST | `/auth/register` | Công khai | A01 |
| POST | `/auth/login` | Công khai | A02 |
| POST | `/auth/refresh` | Cookie refresh token | A03 |
| POST | `/auth/logout` | Đã đăng nhập | A04 |
| GET | `/auth/me` | Đã đăng nhập | A02 |
| GET | `/users` | `users:read` | A06 |
| POST | `/users` | `users:create` | A07 |
| GET | `/users/:id` | `users:read` hoặc chính chủ | A06 |
| PATCH | `/users/me`, `/users/me/password` | Đã đăng nhập | A05 |
| PATCH | `/users/:id/status` | `users:lock` | A08 |
| PUT | `/users/:id/roles` | `users:update` | A09 |
| GET | `/roles`, `/permissions` | `roles:manage` | A10 |
| PUT | `/roles/:id/permissions` | `roles:manage` | A10 |

### Courses (E2)

| Method | Endpoint | Quyền | Story |
|---|---|---|---|
| POST | `/courses` | `courses:create` | C01 |
| GET | `/courses` | `courses:read`; `?managed=true` cho khoá mình quản lý (ADMIN: tất cả) | C07, C02 |
| GET | `/courses/:id` | Khoá đã công bố (kèm `myEnrollment`, `seatsLeft`); khoá Nháp: quản lý khoá | C07, C02 |
| PATCH | `/courses/:id` | Quản lý khoá | C02 |
| PATCH | `/courses/:id/status` | Quản lý khoá | C03 |
| POST | `/courses/:id/duplicate` | Quản lý khoá | C09 |
| GET | `/courses/:courseId/chapters` | Đề cương (tên chương/bài, không kèm nội dung) | C07, C04 |
| POST | `/courses/:courseId/chapters` | Quản lý khoá | C04 |
| PATCH, DELETE | `/chapters/:id` | Quản lý khoá | C04 |
| PUT | `/courses/:courseId/outline` | Quản lý khoá; thứ tự chương và bài, kể cả chuyển bài giữa các chương | C04, C05 |
| POST | `/chapters/:chapterId/lessons` | Quản lý khoá | C05 |
| GET | `/lessons/:id` | Quản lý khoá hoặc học viên APPROVED (kèm nội dung) | C08 |
| PATCH, DELETE | `/lessons/:id` | Quản lý khoá | C05 |

### Enrollments (E3)

| Method | Endpoint | Quyền | Story |
|---|---|---|---|
| POST | `/courses/:courseId/enrollments` | `enrollments:request` | E01 |
| GET | `/courses/:courseId/enrollments` | Quản lý khoá; `?status=` | E02, E05 |
| PATCH | `/enrollments/:id/status` | Quản lý khoá; `{ status: APPROVED\|REJECTED }` | E02 |
| POST | `/courses/:courseId/students` | Quản lý khoá; `{ email }` | E03 |
| GET | `/me/enrollments` | Đã đăng nhập (học viên) | E04 |

### Progress (E4)

| Method | Endpoint | Quyền | Story |
|---|---|---|---|
| PUT | `/lessons/:lessonId/progress` | `progress:track`; `{ completed: boolean }` | P01 |
| GET | `/courses/:courseId/progress/me` | Học viên APPROVED | P02 |
| GET | `/courses/:courseId/progress` | `progress:read-class`, quản lý khoá | P03 |

### Assignments (E5)

| Method | Endpoint | Quyền | Story |
|---|---|---|---|
| POST | `/courses/:courseId/assignments` | `assignments:manage`, quản lý khoá | S01 |
| GET | `/courses/:courseId/assignments` | Quản lý khoá hoặc học viên APPROVED (học viên kèm trạng thái của mình) | S02 |
| GET | `/assignments/:id` | Như trên (học viên kèm `mySubmission`) | S02, S08 |
| PATCH, DELETE | `/assignments/:id` | Quản lý khoá | S01 |
| PUT | `/assignments/:id/submission` | `assignments:submit`; nộp và nộp lại (`{ linkUrl, note }`) | S03, S05 |
| GET | `/assignments/:id/submissions` | Quản lý khoá; `?status=&search=` | S06 |
| PUT | `/submissions/:id/grade` | `submissions:grade`; `{ score, feedback }` | S07 |
| GET | `/courses/:courseId/grades/me` | Học viên APPROVED | S08 |
| GET | `/courses/:courseId/grades/export?format=csv` | `grades:export`, quản lý khoá | S09 |

### Khác

| Method | Endpoint | Ghi chú |
|---|---|---|
| GET | `/health` | Công khai, không cần xác thực, kiểm tra kết nối DB |
| (dự trữ) | `/me/notifications` | Phase 2 (US-S11) |

## 9. Quy ước cho frontend (hệ quả của tài liệu này)

- Một lớp HTTP duy nhất: gắn access token, xử lý 401 bằng **một lần** refresh rồi gửi lại, nếu thất bại thì đưa về đăng nhập.
- `code` lỗi ánh xạ sang thông báo tại một file (`shared/lib/errorMessages.ts`), không rải chuỗi trong component.
- `errors[]` của `VALIDATION_FAILED` gắn thẳng vào trường form tương ứng.
- Hiển thị thời gian chỉ qua `formatDateTime()` (GMT+7, `dd/mm/yyyy HH:mm`).

## 10. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 0.1 | 09/10/2026 | Bản nháp đầu tiên |
| 0.2 | 09/10/2026 | Thêm mã lỗi `ROLE_PROTECTED_PERMISSION`, `AUTH_CURRENT_PASSWORD_WRONG` (đồng bộ với `auth-spec.md` v1.1) |
| 0.3 | 10/10/2026 | Thêm mã `BAD_REQUEST`, `SERVICE_UNAVAILABLE`; ghi quy tắc `X-Request-Id` và phân biệt lỗi framework / lỗi nghiệp vụ (đồng bộ với lớp `common` của backend) |
