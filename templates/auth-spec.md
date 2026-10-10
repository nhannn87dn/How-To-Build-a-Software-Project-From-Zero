# Spec: Module Auth (Users, Roles, Permissions)

> Phiên bản: 1.1 | Ngày: 09/10/2026 | Trạng thái: Draft
> Yêu cầu: `auth-prd.md` | Story: `auth-user-stories.md`
> Mọi endpoint có tiền tố `/api/v1`; mã lỗi nội bộ và cấu trúc response theo `03-design/api-conventions.md`.

---

## 1. Database (SQL Server)

```
users
  id              BIGINT IDENTITY PK
  email           NVARCHAR(255) NOT NULL UNIQUE      -- lưu chữ thường
  password_hash   NVARCHAR(255) NOT NULL
  full_name       NVARCHAR(100) NOT NULL
  status          VARCHAR(20)   NOT NULL DEFAULT 'ACTIVE'   -- ACTIVE | LOCKED
  created_at      DATETIME2     NOT NULL DEFAULT SYSUTCDATETIME()
  updated_at      DATETIME2     NOT NULL DEFAULT SYSUTCDATETIME()

roles
  id              INT IDENTITY PK
  code            VARCHAR(30)   NOT NULL UNIQUE      -- ADMIN | INSTRUCTOR | STUDENT
  name            NVARCHAR(100) NOT NULL

permissions
  id              INT IDENTITY PK
  code            VARCHAR(100)  NOT NULL UNIQUE      -- ví dụ users:read
  description     NVARCHAR(255) NULL

user_roles
  user_id         BIGINT NOT NULL FK -> users(id)  ON DELETE CASCADE
  role_id         INT    NOT NULL FK -> roles(id)
  PK (user_id, role_id)
  INDEX (role_id)

role_permissions
  role_id         INT NOT NULL FK -> roles(id)       ON DELETE CASCADE
  permission_id   INT NOT NULL FK -> permissions(id) ON DELETE CASCADE
  PK (role_id, permission_id)
  INDEX (permission_id)

refresh_tokens
  id              BIGINT IDENTITY PK
  user_id         BIGINT   NOT NULL FK -> users(id) ON DELETE CASCADE
  token_hash      CHAR(64) NOT NULL UNIQUE           -- SHA-256, không lưu token gốc
  expires_at      DATETIME2 NOT NULL
  revoked_at      DATETIME2 NULL
  replaced_by_id  BIGINT    NULL                     -- trỏ tới token mới khi xoay vòng
  created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
  INDEX (user_id, revoked_at)
```

**Index bổ sung:** `users(status)`, `users(full_name)` phục vụ lọc/tìm kiếm.

**Seed (migration riêng):**
- 3 role: ADMIN, INSTRUCTOR, STUDENT
- Danh sách permission (mục 3) và ánh xạ mặc định role -> permission
- 1 tài khoản ADMIN đầu tiên từ biến môi trường `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD`

## 2. API

Quy ước: response `{ success, data, message }`; danh sách trả `{ items, page, limit, total }` trong `data`.

### 2.1 Auth

| Method | Endpoint | Quyền | Body | Thành công | Lỗi |
|---|---|---|---|---|---|
| POST | `/auth/register` | Public | `email, password, fullName` | 201 user (không có password) | 400 `VALIDATION_FAILED`, 409 `USER_EMAIL_TAKEN` |
| POST | `/auth/login` | Public | `email, password` | 200 `{ accessToken, user }` và đặt cookie refresh token | 400, 401 `AUTH_INVALID_CREDENTIALS`, 403 `AUTH_ACCOUNT_LOCKED`, 429 `RATE_LIMITED` |
| POST | `/auth/refresh` | Cookie refresh token | (không có body) | 200 `{ accessToken }` và đặt cookie refresh token mới | 401 `AUTH_REFRESH_INVALID`, 403 `FORBIDDEN` (Origin lạ) |
| POST | `/auth/logout` | Cookie refresh token | (không có body) | 200, thu hồi token và xoá cookie | 401, 403 `FORBIDDEN` (Origin lạ) |
| GET | `/auth/me` | Đã đăng nhập | | 200 user + roles | 401 `AUTH_TOKEN_INVALID` |

### 2.1.1 Cookie refresh token (ADR-006)

| Thuộc tính | Giá trị |
|---|---|
| Tên | `refresh_token` |
| `HttpOnly` | Có (JavaScript không đọc được) |
| `Secure` | Có ở production; tắt được bằng `COOKIE_SECURE=false` khi phát triển trên http |
| `SameSite` | `Strict` |
| `Path` | `/api/v1/auth` (cookie chỉ được gửi tới nhóm endpoint auth) |
| Thời hạn | `JWT_REFRESH_EXPIRES_DAYS` ngày |

- Access token trả trong body và chỉ được giữ trong bộ nhớ của frontend
- `/auth/refresh` và `/auth/logout` kiểm tra header `Origin` phải trùng `FRONTEND_ORIGIN`, nếu không trả 403
- CORS bật `credentials`, chỉ cho đúng `FRONTEND_ORIGIN`
- Khi tải lại trang, frontend gọi `/auth/refresh` để khôi phục phiên

### 2.2 Users

| Method | Endpoint | Quyền | Mô tả | Lỗi |
|---|---|---|---|---|
| GET | `/users` | `users:read` | Query: `page, limit, search, role, status` | 401, 403 |
| GET | `/users/:id` | `users:read` hoặc chính chủ | Chi tiết user | 403, 404 |
| POST | `/users` | `users:create` | Body: `email, fullName, password, roleCodes[]` | 400, 409 `USER_EMAIL_TAKEN` |
| PATCH | `/users/me` | Đã đăng nhập | Body: `fullName` | 400 |
| PATCH | `/users/me/password` | Đã đăng nhập | Body: `currentPassword, newPassword` | 400 `VALIDATION_FAILED` hoặc `AUTH_CURRENT_PASSWORD_WRONG` |
| PATCH | `/users/:id` | `users:update` | Body: `fullName` | 404 |
| PATCH | `/users/:id/status` | `users:lock` | Body: `status` | 409 `USER_CANNOT_LOCK_SELF`, 404 |
| PUT | `/users/:id/roles` | `users:update` | Body: `roleCodes[]` | 409 `USER_LAST_ADMIN`, 404 |

### 2.3 Roles & Permissions

| Method | Endpoint | Quyền | Mô tả | Lỗi |
|---|---|---|---|---|
| GET | `/roles` | `roles:manage` | Danh sách role kèm permission | 403 |
| GET | `/permissions` | `roles:manage` | Toàn bộ permission | 403 |
| PUT | `/roles/:id/permissions` | `roles:manage` | Body: `permissionCodes[]` | 409 `ROLE_PROTECTED_PERMISSION`, 404 |

> `GET /users/:id` và `GET /auth/me` dùng chung DTO response, luôn loại bỏ `passwordHash`.

## 3. Permission (Phase 1, phần thuộc module auth)

| Permission | ADMIN | INSTRUCTOR | STUDENT |
|---|:-:|:-:|:-:|
| `users:read` | ✓ | | |
| `users:create` | ✓ | | |
| `users:update` | ✓ | | |
| `users:lock` | ✓ | | |
| `roles:manage` | ✓ | | |

Module sau tự bổ sung permission qua migration seed riêng, theo quy tắc `<resource>:<action>`. Danh mục permission đầy đủ của các module: `03-design/architecture.md` mục 7.1.

## 4. Luồng xử lý chính

**Đăng nhập:**
1. Tìm user theo email đã chuẩn hoá chữ thường
2. So khớp bcrypt; sai thì trả 401 chung (email không tồn tại vẫn so khớp với hash giả để thời gian phản hồi tương đương)
3. Kiểm tra `status = ACTIVE`, nếu không trả 403
4. Tạo access token (payload: `sub`, `roles`), tạo refresh token ngẫu nhiên 64 byte, lưu SHA-256 của nó
5. Trả access token và thông tin user trong body; đặt refresh token vào cookie (xem 2.1.1)

**Refresh (rotation + reuse detection):**
0. Kiểm tra `Origin` trùng `FRONTEND_ORIGIN`, nếu không trả 403
1. Đọc refresh token từ cookie, băm rồi tìm trong `refresh_tokens`
2. Không thấy hoặc hết hạn: 401
3. Đã có `revoked_at`: nghi bị đánh cắp, thu hồi mọi token của user, trả 401
4. Hợp lệ: đánh dấu `revoked_at`, tạo refresh token mới, ghi `replaced_by_id`, trả access token mới trong body và đặt cookie refresh token mới

**Kiểm tra quyền (2 lớp):**
- Guard: `JwtAuthGuard` xác thực token; `PermissionsGuard` đọc decorator `@RequirePermissions('users:read')`, tra permission của các role trong token từ cache
- Service: kiểm tra theo chủ sở hữu (ví dụ không có `users:read` thì chỉ xem được `/users/:id` của chính mình)

## 5. Edge case & ràng buộc

- Email chuẩn hoá `trim().toLowerCase()` trước khi lưu và so sánh
- Khoá user: trong cùng transaction, đặt `status = LOCKED` và thu hồi mọi refresh token; `JwtStrategy` kiểm tra thêm `status` để chặn ngay access token còn sống
- ValidationPipe `whitelist: true, forbidNonWhitelisted: true` để chặn client nhét thêm trường (ví dụ `roles` khi đăng ký)
- Không log mật khẩu hoặc token
- Dọn refresh token hết hạn bằng job định kỳ (có thể làm sau)
- Nhiều tab cùng gọi `/auth/refresh` với một cookie: tab thứ hai sẽ bị coi là dùng lại token. Phía frontend chỉ cho một lời gọi refresh đang chạy mỗi lúc và đồng bộ giữa các tab; nếu vẫn thấy người dùng bị đăng xuất oan thì thêm khoảng "đợi ngắn" ở server (ADR-006)

## 6. Cấu hình môi trường (`.env.example`)

```
DB_HOST=localhost
DB_PORT=1433
DB_USERNAME=
DB_PASSWORD=
DB_NAME=lms
JWT_ACCESS_SECRET=
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_DAYS=7
BCRYPT_COST=10
SEED_ADMIN_EMAIL=
SEED_ADMIN_PASSWORD=
FRONTEND_ORIGIN=http://localhost:5173
COOKIE_SECURE=false
```

## 7. Cấu trúc thư mục dự kiến

```
src/
  auth/         module, controller, service, strategies, guards, decorators, dto
  users/        module, controller, service, dto, entities
  roles/        module, controller, service, dto, entities (gồm permissions)
  common/       exception filter, response interceptor, pagination dto
  database/     migrations, seeds
```

## 8. Kế hoạch test (Jest, map từ AC)

| Mã | Story | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-01 | US-A01 | Đăng ký kèm `roles: ['ADMIN']` | 400 hoặc bỏ qua, user có role STUDENT |
| T-02 | US-A01 | Đăng ký email đã tồn tại (khác hoa/thường) | 409 |
| T-03 | US-A02 | Sai mật khẩu và email không tồn tại | Cùng 401, cùng message |
| T-04 | US-A02 | Đăng nhập tài khoản LOCKED | 403 |
| T-05 | US-A02 | Đăng nhập sai 6 lần trong 1 phút | Lần thứ 6 trả 429 |
| T-06 | US-A03 | Refresh hợp lệ (cookie còn hạn) | Access token mới và cookie refresh token mới; token cũ bị thu hồi |
| T-07 | US-A03 | Dùng lại refresh token đã dùng | 401, toàn bộ token của user bị thu hồi |
| T-08 | US-A04 | Logout rồi gọi refresh với cookie cũ | 401, cookie đã bị xoá |
| T-09 | US-A05 | Đổi mật khẩu thành công | Mọi refresh token cũ bị thu hồi |
| T-10 | US-A06 | STUDENT gọi `GET /users` | 403 |
| T-11 | US-A05 | STUDENT gọi `GET /users/{id người khác}` | 403 |
| T-12 | US-A08 | ADMIN khoá chính mình | 409 `USER_CANNOT_LOCK_SELF` |
| T-13 | US-A08 | Khoá user đang đăng nhập | Request kế tiếp bằng access token cũ bị 401 |
| T-14 | US-A09 | Gỡ role ADMIN của admin cuối cùng | 409 `USER_LAST_ADMIN` |
| T-15 | US-A10 | Đổi permission của role rồi gọi API | Có hiệu lực sau khi cache hết hạn/bị xoá |
| T-16 | US-A03 | Gọi `/auth/refresh` hoặc `/auth/logout` với `Origin` lạ | 403 |
| T-17 | US-A02 | Kiểm tra cookie sau đăng nhập | Có `HttpOnly`, `SameSite=Strict`, `Path=/api/v1/auth` (và `Secure` ở production); access token không nằm trong cookie |

Ưu tiên viết trước: T-02, T-03, T-06, T-07, T-12, T-14.

## 9. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 09/10/2026 | Bản đầu tiên |
| 1.1 | 09/10/2026 | Refresh token chuyển từ body sang cookie HttpOnly (ADR-006) kèm kiểm tra Origin; thêm tiền tố `/api/v1`; mã lỗi nội bộ theo `api-conventions.md` (khoá mình, admin cuối, quyền ADMIN trả 409); thêm biến `FRONTEND_ORIGIN`, `COOKIE_SECURE`; thêm T-16, T-17 |
