# Module: Auth (Users, Roles, Permissions)

> Phiên bản: 1.1 | Ngày: 09/10/2026 | Trạng thái: Draft
> Phụ thuộc: không. Các module courses, enrollments, lessons phụ thuộc module này.
> Mã yêu cầu của dự án: US-A01 đến US-A10 tương ứng FR-AUTH-01 đến FR-AUTH-06 (xem `02-user-stories.md`). Mọi endpoint có tiền tố `/api/v1`; mã lỗi và cấu trúc response theo `03-design/api-conventions.md`.

---

## 1. Mục tiêu & phạm vi (PRD thu nhỏ)

**Vì sao:** Hệ thống LMS có 3 nhóm người dùng với quyền khác nhau. Cần xác thực danh tính và phân quyền nhất quán làm nền cho mọi module khác.

**Trong scope (Phase 1):**
- Đăng ký (chỉ STUDENT tự đăng ký), đăng nhập, đăng xuất, làm mới token
- Quản lý user (ADMIN): tạo, xem, sửa, khoá/mở khoá
- Gán role cho user, gán permission cho role (ADMIN)
- Đổi mật khẩu, xem/sửa hồ sơ cá nhân

**Ngoài scope (để Phase sau):**
- Quên mật khẩu qua email, xác thực email
- Đăng nhập Google/Facebook (OAuth)
- 2FA
- Tạo role mới tuỳ ý qua giao diện (Phase 1 chỉ 3 role cố định, seed sẵn)

**Yêu cầu phi chức năng:**
- Mật khẩu băm bằng bcrypt (cost 10-12), không bao giờ trả ra client
- Access token sống ngắn (15 phút), refresh token 7 ngày, có xoay vòng (rotation)
- Rate limit endpoint đăng nhập: tối đa 5 lần/phút/IP
- Thông báo lỗi đăng nhập không tiết lộ email có tồn tại hay không

**Giả định (cần bạn xác nhận):**
- Một user có thể có **nhiều role** (ví dụ giảng viên vừa dạy vừa học khoá khác), nên dùng bảng nối `user_roles`
- Permission là danh sách mã cố định do dev khai báo trong code và seed vào DB; ADMIN chỉ gán/bỏ gán, không tạo permission mới
- Role và permission được đưa vào JWT chỉ ở mức **role**; permission của role được tra từ cache trong bộ nhớ, tránh token phình to và tránh quyền cũ sau khi ADMIN đổi

---

## 2. User Stories

### Xác thực (mọi người dùng)

**US-A01:** Với vai trò khách, tôi muốn đăng ký tài khoản để học các khoá học.
- AC:
  - Nhập email, mật khẩu, họ tên; email chưa tồn tại
  - Mật khẩu tối thiểu 8 ký tự, có chữ hoa, chữ thường và số
  - Tài khoản tạo ra luôn có role STUDENT, bỏ qua mọi trường role gửi lên
  - Email trùng thì báo lỗi rõ ràng

**US-A02:** Với vai trò người dùng đã có tài khoản, tôi muốn đăng nhập để sử dụng hệ thống.
- AC:
  - Đúng email + mật khẩu thì nhận access token và refresh token
  - Sai email hoặc sai mật khẩu đều trả cùng một thông báo chung
  - Tài khoản bị khoá thì không đăng nhập được
  - Quá 5 lần/phút thì bị chặn tạm thời

**US-A03:** Với vai trò người dùng đã đăng nhập, tôi muốn phiên làm việc tự gia hạn để không bị đăng xuất giữa chừng.
- AC:
  - Access token hết hạn thì tự lấy access token mới bằng refresh token trong cookie HttpOnly; refresh token cũng được xoay vòng
  - Mỗi refresh token chỉ dùng được một lần; dùng lại token đã dùng thì thu hồi toàn bộ phiên của user đó
  - Refresh token hết hạn hoặc đã thu hồi thì yêu cầu đăng nhập lại
  - Tải lại trang thì phiên được khôi phục nếu refresh token còn hiệu lực

**US-A04:** Với vai trò người dùng đã đăng nhập, tôi muốn đăng xuất để kết thúc phiên.
- AC:
  - Refresh token hiện tại bị thu hồi ngay
  - Dùng lại token đó để refresh sẽ bị từ chối

**US-A05:** Với vai trò người dùng đã đăng nhập, tôi muốn xem và sửa hồ sơ, đổi mật khẩu của mình.
- AC:
  - Xem được thông tin cá nhân (không có mật khẩu)
  - Chỉ sửa được họ tên; không tự đổi email, role, status
  - Đổi mật khẩu phải nhập đúng mật khẩu cũ
  - Đổi mật khẩu xong, mọi refresh token cũ bị thu hồi

### Quản trị (ADMIN)

**US-A06:** Với vai trò ADMIN, tôi muốn xem danh sách user có tìm kiếm, lọc, phân trang để quản lý.
- AC:
  - Lọc theo role, status; tìm theo email hoặc họ tên
  - Phân trang `page`, `limit` (mặc định 1 và 20, tối đa 100)

**US-A07:** Với vai trò ADMIN, tôi muốn tạo tài khoản giảng viên để giảng viên bắt đầu tạo khoá học.
- AC:
  - Nhập email, họ tên, mật khẩu tạm, danh sách role
  - Email không được trùng

**US-A08:** Với vai trò ADMIN, tôi muốn khoá/mở khoá tài khoản để xử lý vi phạm.
- AC:
  - Khoá xong, user không đăng nhập được và mọi refresh token bị thu hồi
  - ADMIN không tự khoá chính mình

**US-A09:** Với vai trò ADMIN, tôi muốn gán role cho user để cấp quyền phù hợp.
- AC:
  - Thay toàn bộ danh sách role của user bằng danh sách mới
  - Không được gỡ role ADMIN của quản trị viên cuối cùng

**US-A10 (Could):** Với vai trò ADMIN, tôi muốn xem và chỉnh permission của từng role để kiểm soát chức năng.
- AC:
  - Xem được danh sách permission và permission của mỗi role
  - Thay danh sách permission của role; thay đổi có hiệu lực sau tối đa 1 phút (hết hạn cache) hoặc ngay lập tức khi cache bị xoá
  - Không được gỡ permission quản trị khỏi role ADMIN (`roles:manage`, `users:update`)

---

## 3. Spec kỹ thuật

### 3.1 Database (SQL Server)

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
  token_hash      CHAR(64) NOT NULL UNIQUE           -- SHA-256 của token, không lưu token gốc
  expires_at      DATETIME2 NOT NULL
  revoked_at      DATETIME2 NULL
  replaced_by_id  BIGINT    NULL                     -- trỏ tới token mới khi xoay vòng
  created_at      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
  INDEX (user_id, revoked_at)
```

**Index bổ sung:** `users(status)`, `users(full_name)` phục vụ lọc/tìm kiếm.

**Seed (migration riêng):**
- 3 role: ADMIN, INSTRUCTOR, STUDENT
- Danh sách permission (mục 3.3) và ánh xạ mặc định role -> permission
- 1 tài khoản ADMIN đầu tiên lấy từ biến môi trường `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD`

### 3.2 API

Quy ước: response `{ success, data, message }`, danh sách có `{ items, page, limit, total }` trong `data`.

#### Auth

| Method | Endpoint | Quyền | Body | Thành công | Lỗi |
|---|---|---|---|---|---|
| POST | `/auth/register` | Public | `email, password, fullName` | 201 user (không có password) | 400 `VALIDATION_FAILED`, 409 `USER_EMAIL_TAKEN` |
| POST | `/auth/login` | Public | `email, password` | 200 `{ accessToken, user }` và đặt cookie refresh token | 400, 401 `AUTH_INVALID_CREDENTIALS`, 403 `AUTH_ACCOUNT_LOCKED`, 429 `RATE_LIMITED` |
| POST | `/auth/refresh` | Cookie refresh token | (không có body) | 200 `{ accessToken }` và đặt cookie refresh token mới | 401 `AUTH_REFRESH_INVALID`, 403 `FORBIDDEN` (Origin lạ) |
| POST | `/auth/logout` | Cookie refresh token | (không có body) | 200, thu hồi token và xoá cookie | 401, 403 `FORBIDDEN` (Origin lạ) |
| GET | `/auth/me` | Đã đăng nhập | | 200 user + roles | 401 `AUTH_TOKEN_INVALID` |

**Cookie refresh token (ADR-006):** tên `refresh_token`; `HttpOnly`; `Secure` (có ở production, tắt được bằng `COOKIE_SECURE=false` khi phát triển trên http); `SameSite=Strict`; `Path=/api/v1/auth`; thời hạn `JWT_REFRESH_EXPIRES_DAYS` ngày. Access token trả trong body và chỉ được giữ trong bộ nhớ của frontend. `/auth/refresh` và `/auth/logout` kiểm tra header `Origin` phải trùng `FRONTEND_ORIGIN`, nếu không trả 403. CORS bật `credentials`, chỉ cho đúng `FRONTEND_ORIGIN`. Khi tải lại trang, frontend gọi `/auth/refresh` để khôi phục phiên.

#### Users

| Method | Endpoint | Quyền | Mô tả | Lỗi |
|---|---|---|---|---|
| GET | `/users` | `users:read` | Query: `page, limit, search, role, status` | 401, 403 |
| GET | `/users/:id` | `users:read` hoặc chính chủ | Chi tiết user | 403, 404 |
| POST | `/users` | `users:create` | Body: `email, fullName, password, roleCodes[]` | 400, 409 |
| PATCH | `/users/me` | Đã đăng nhập | Body: `fullName` | 400 |
| PATCH | `/users/me/password` | Đã đăng nhập | Body: `currentPassword, newPassword` | 400 `VALIDATION_FAILED` hoặc `AUTH_CURRENT_PASSWORD_WRONG` |
| PATCH | `/users/:id` | `users:update` | Body: `fullName` | 404 |
| PATCH | `/users/:id/status` | `users:lock` | Body: `status` (ACTIVE/LOCKED) | 409 `USER_CANNOT_LOCK_SELF`, 404 |
| PUT | `/users/:id/roles` | `users:update` | Body: `roleCodes[]` | 409 `USER_LAST_ADMIN`, 404 |

#### Roles & Permissions

| Method | Endpoint | Quyền | Mô tả | Lỗi |
|---|---|---|---|---|
| GET | `/roles` | `roles:manage` | Danh sách role kèm permission | 403 |
| GET | `/permissions` | `roles:manage` | Danh sách toàn bộ permission | 403 |
| PUT | `/roles/:id/permissions` | `roles:manage` | Body: `permissionCodes[]` | 409 `ROLE_PROTECTED_PERMISSION`, 404 |

> `GET /users/:id` và `GET /auth/me` dùng chung một DTO response, luôn loại bỏ `passwordHash`.

### 3.3 Permission (Phase 1, chỉ phần thuộc module auth)

| Permission | ADMIN | INSTRUCTOR | STUDENT |
|---|:-:|:-:|:-:|
| `users:read` | ✓ | | |
| `users:create` | ✓ | | |
| `users:update` | ✓ | | |
| `users:lock` | ✓ | | |
| `roles:manage` | ✓ | | |

Các module sau (courses, lessons, enrollments) tự bổ sung permission của mình qua migration seed riêng, theo quy tắc đặt tên `<resource>:<action>` (ví dụ `courses:create`, `courses:publish`).

### 3.4 Luồng xử lý chính

**Đăng nhập:**
1. Tìm user theo email (đã chuẩn hoá chữ thường)
2. So khớp bcrypt; sai thì trả 401 chung (kể cả khi email không tồn tại, vẫn nên so khớp với hash giả để thời gian phản hồi tương đương)
3. Kiểm tra `status = ACTIVE`, nếu không trả 403
4. Tạo access token (payload: `sub`, `roles`), tạo refresh token ngẫu nhiên 64 byte, lưu SHA-256 của nó
5. Trả access token và thông tin user trong body; đặt refresh token vào cookie

**Refresh (rotation + reuse detection):**
0. Kiểm tra `Origin` trùng `FRONTEND_ORIGIN`, nếu không trả 403
1. Đọc refresh token từ cookie, băm rồi tìm trong `refresh_tokens`
2. Không thấy hoặc hết hạn: 401
3. Đã có `revoked_at`: **nghi bị đánh cắp**, thu hồi mọi token của user đó, trả 401
4. Hợp lệ: đánh dấu `revoked_at`, tạo refresh token mới, ghi `replaced_by_id`, trả access token mới trong body và đặt cookie refresh token mới

**Kiểm tra quyền (2 lớp):**
- Guard: `JwtAuthGuard` xác thực token, `PermissionsGuard` đọc decorator `@RequirePermissions('users:read')`, tra permission của các role trong token từ cache
- Service: kiểm tra theo chủ sở hữu (ví dụ user chỉ xem được `GET /users/:id` của chính mình nếu không có `users:read`)

### 3.5 Edge case & ràng buộc

- Email chuẩn hoá `trim().toLowerCase()` trước khi lưu và so sánh
- Khoá user: trong cùng transaction, đặt `status = LOCKED` và thu hồi mọi refresh token; access token còn sống tối đa 15 phút, `JwtStrategy` kiểm tra thêm `status` để chặn ngay
- ValidationPipe `whitelist: true, forbidNonWhitelisted: true` để chặn client nhét thêm trường (ví dụ `roles` khi đăng ký)
- Không bao giờ log mật khẩu hoặc token
- Dọn refresh token hết hạn bằng job định kỳ (có thể làm sau)
- Nhiều tab cùng gọi `/auth/refresh` với một cookie: tab thứ hai sẽ bị coi là dùng lại token. Frontend chỉ cho một lời gọi refresh đang chạy mỗi lúc và đồng bộ giữa các tab; nếu vẫn có người dùng bị đăng xuất oan thì thêm khoảng "đợi ngắn" ở server (ADR-006)

### 3.6 Cấu hình môi trường (`.env.example`)

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

### 3.7 Cấu trúc thư mục dự kiến

```
src/
  auth/         module, controller, service, strategies, guards, decorators, dto
  users/        module, controller, service, dto, entities
  roles/        module, controller, service, dto, entities (gồm permissions)
  common/       exception filter, response interceptor, pagination dto
  database/     migrations, seeds
```

---

## 4. Test cases (map từ AC)

| Mã | Story | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| T-01 | US-A01 | Đăng ký với `roles: ['ADMIN']` trong body | 400 (bị whitelist chặn) hoặc bỏ qua, user có role STUDENT |
| T-02 | US-A01 | Đăng ký email đã tồn tại (khác hoa/thường) | 409 |
| T-03 | US-A02 | Sai mật khẩu và email không tồn tại | Cùng 401, cùng message |
| T-04 | US-A02 | Đăng nhập tài khoản LOCKED | 403 |
| T-05 | US-A02 | Đăng nhập sai 6 lần trong 1 phút | Lần thứ 6 trả 429 |
| T-06 | US-A03 | Refresh hợp lệ (cookie còn hạn) | Access token mới và cookie refresh token mới; token cũ bị thu hồi |
| T-07 | US-A03 | Dùng lại refresh token đã dùng | 401 và toàn bộ token của user bị thu hồi |
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

> Ưu tiên viết Jest cho: T-02, T-03, T-06, T-07, T-12, T-14 (logic quan trọng nhất của auth).

---

## 5. Câu hỏi còn mở

1. Một user nhiều role có đúng ý bạn không, hay chỉ một role mỗi user? *(giả định đang dùng: nhiều role)*
2. Có cho phép ADMIN chỉnh permission của role qua giao diện ở Phase 1, hay chỉ seed cố định (đơn giản hơn)? *(theo PRD dự án: Could, không thuộc nhóm Must của Phase 1)*
3. Cần xoá user thật sự hay chỉ khoá (khuyến nghị: chỉ khoá, vì còn dữ liệu đăng ký học, bài nộp)? *(giả định đang dùng: chỉ khoá)*

---

## 6. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 09/10/2026 | Bản đầu tiên |
| 1.1 | 09/10/2026 | Refresh token chuyển sang cookie HttpOnly kèm kiểm tra Origin (ADR-006); thêm tiền tố `/api/v1`; mã lỗi nội bộ theo `api-conventions.md` (khoá mình, admin cuối, quyền ADMIN trả 409); thêm biến `FRONTEND_ORIGIN`, `COOKIE_SECURE`; thêm T-16, T-17; US-A10 là Could; ghi chú mã `FR-AUTH-xx` |
