# Kiến trúc hệ thống LMS

> Phiên bản: 0.4 (Draft) | Ngày: 10/10/2026
> Nguồn: `01-prd.md`, `02-user-stories.md` | Liên quan: `erd.md`, `api-conventions.md`, `ui-flows.md`, `adr/`
> Nhãn **[ĐÃ CHỐT]** = quyết định đã được xác nhận ngày 09/10/2026 (có ADR đi kèm).

---

## 1. Phạm vi và giả định thiết kế

- Thiết kế cho **Phase 1 (MVP)**, giả định đã hoãn US-C06 (đính kèm tệp), US-S04 (nộp bài bằng tệp), US-E06 (gỡ học viên). Vì vậy **chưa có hạ tầng lưu tệp**.
- Quy mô: 1-2 ADMIN, 2-5 giảng viên, 100-200 tài khoản học viên, tối đa ~100 người dùng đồng thời.
- Một người phát triển, hạ tầng nhỏ (một máy chủ hoặc PaaS nhỏ).
- Một khoá = một đợt học (ADR-002). Tiến độ và điểm tổng được tính khi cần, không lưu (ADR-003).

## 2. Kiến trúc tổng thể: modular monolith (ADR-001)

```
 Trình duyệt (React SPA, TypeScript)
        │  HTTPS, JSON (REST), prefix /api/v1
        ▼
 ┌──────────────────────────────────────────────────────┐
 │ NestJS: MỘT ứng dụng, chia module                    │
 │                                                      │
 │  auth   users   roles        (E1)                    │
 │  courses                      (E2)                   │
 │  enrollments                  (E3)                   │
 │  progress                     (E4)                   │
 │  assignments (+ chấm điểm, xuất điểm)   (E5)         │
 │  ─────────────────────────────────────────────       │
 │  course-access  (quyền và quan hệ khoá–học viên)     │
 │  common (filter, interceptor, guard, pipe, pagination)│
 │  config, database (migration, seed), health          │
 └───────────────────────┬──────────────────────────────┘
                         ▼
                    SQL Server
```

Chỉ có **một process backend, một database**. Ranh giới giữa các module được giữ bằng quy tắc ở mục 4 chứ không bằng mạng.

## 3. Bản đồ module

| Module | Epic | Trách nhiệm | Entity sở hữu (chỉ module này được ghi) |
|---|---|---|---|
| `auth` | E1 | Đăng ký, đăng nhập, gia hạn phiên, đăng xuất | RefreshToken |
| `users` | E1 | Quản lý user, hồ sơ, khoá/mở khoá, gán role | User, UserRole |
| `roles` | E1 | Role, permission, gán permission cho role | Role, Permission, RolePermission |
| `courses` | E2 | Khoá, chương, bài học, sắp xếp outline, trạng thái, nhân bản | Course, Chapter, Lesson |
| `enrollments` | E3 | Đăng ký, duyệt, thêm trực tiếp, danh sách lớp | Enrollment |
| `progress` | E4 | Đánh dấu hoàn thành, tiến độ cá nhân và cả lớp | LessonProgress |
| `assignments` | E5 | Bài tập, bài nộp, chấm điểm, bảng điểm, xuất CSV | Assignment, Submission |
| `course-access` | Mọi epic | Trả lời "ai được làm gì trên khoá nào" và các số liệu quan hệ khoá–học viên (chỉ đọc) | (đọc Course, Enrollment) |
| `common` | | Exception filter, response interceptor, guard, decorator, pagination | |
| `config`, `database`, `health` | | Đọc/kiểm tra biến môi trường; data source, migration, seed; kiểm tra sức khoẻ | |

## 4. Quy tắc phụ thuộc giữa module

**Quy tắc:**
1. Mỗi entity có **đúng một module sở hữu**; chỉ module đó ghi vào bảng của nó.
2. Module khác cần dữ liệu thì gọi **service được export** của module sở hữu, không truy vấn thẳng bảng của module khác. Ngoại lệ duy nhất: `course-access` đọc Course và Enrollment (chỉ đọc).
3. **Cấm phụ thuộc vòng.**
4. Controller không chứa logic nghiệp vụ; service không biết gì về HTTP (không import Request/Response).

| Module | Được phép phụ thuộc |
|---|---|
| `auth` | `users`, `roles` |
| `users` | `roles` |
| `roles` | (không) |
| `course-access` | (chỉ entity Course, Enrollment) |
| `courses` | `course-access`, `users` |
| `enrollments` | `course-access`, `courses`, `users` |
| `progress` | `course-access`, `courses`, `enrollments` |
| `assignments` | `course-access`, `courses`, `enrollments`, `users` |

**Vì sao có `course-access`:** `courses` cần biết "còn bao nhiêu chỗ" và "học viên này đang ở trạng thái đăng ký nào" (US-C07, US-C08) nhưng `enrollments` đã phụ thuộc `courses`; nếu `courses` gọi ngược lại sẽ thành vòng. Đặt các truy vấn chỉ đọc đó, cùng các hàm kiểm tra quyền, vào một module trung lập giải quyết được cả hai.

Hàm chính của `CourseAccessService` (chữ ký dự kiến):

```ts
// src/course-access/course-access.service.ts
assertCanManage(courseId: number, user: AuthUser): Promise<Course>   // chủ khoá hoặc có courses:manage-any, nếu không: 403 (hoặc 404 với khoá Nháp của người khác)
assertCanLearn(courseId: number, user: AuthUser): Promise<Course>    // học viên APPROVED, khoá PUBLISHED hoặc ENDED
getApprovedCounts(courseIds: number[]): Promise<Map<number, number>> // gộp một truy vấn, tránh N+1
getEnrollmentStatuses(userId: number, courseIds: number[]): Promise<Map<number, EnrollmentStatus>>
```

## 5. Cấu trúc thư mục

### Backend

```
src/
  main.ts  app.module.ts
  config/                 đọc và kiểm tra biến môi trường khi khởi động
  common/                 filters/ interceptors/ guards/ decorators/ dto/pagination.dto.ts utils/
  database/               data-source.ts  migrations/  seeds/
  course-access/          module, service
  auth/ users/ roles/ courses/ enrollments/ progress/ assignments/
    <module>.module.ts  <module>.controller.ts  <module>.service.ts
    dto/  entities/  (và <module>.service.spec.ts cạnh file service)
  health/
test/                     e2e (supertest)
```

### Frontend

```
src/
  app/                    router, providers (QueryClient, Auth), layouts
  features/
    auth/ users/ courses/ enrollments/ learning/ assignments/
      api/                hàm gọi API của tính năng (lớp service)
      hooks/              useCourses, useEnroll ... (bọc data fetching)
      components/  pages/  schemas/ (Zod)  types.ts
  shared/
    components/           AppShell, DataTable, Pagination, EmptyState, ErrorState,
                          LoadingState, ConfirmDialog, FormField, StatusBadge
    lib/                  http client, token store, error mapping (code -> thông báo)
    utils/                formatDateTime (GMT+7, dd/mm/yyyy HH:mm)
```

## 6. Xác thực (authentication)

| Hạng mục | Thiết kế |
|---|---|
| Access token | JWT, sống 15 phút, payload `{ sub, roles }`, ký bằng secret trong biến môi trường |
| Refresh token | Chuỗi ngẫu nhiên 64 byte, sống 7 ngày, lưu **SHA-256 hash** trong DB, xoay vòng mỗi lần dùng, phát hiện dùng lại thì thu hồi mọi phiên |
| Nơi lưu ở trình duyệt | **[ĐÃ CHỐT, ADR-006]** access token trong bộ nhớ (biến JS); refresh token trong cookie `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth` |
| Khôi phục sau khi tải lại trang | Frontend gọi `POST /auth/refresh` (cookie tự đính kèm) để lấy lại access token |
| Kiểm tra trạng thái | `JwtStrategy` tra user theo khoá chính mỗi request để chặn ngay tài khoản bị khoá (một truy vấn theo PK, chấp nhận được ở quy mô này) |
| Mật khẩu | bcrypt, cost cấu hình qua `BCRYPT_COST` |
| Chống dò mật khẩu | Rate limit `POST /auth/login`: 5 lần/phút/IP |

> `auth-spec.md` (v1.1) đã được cập nhật theo ADR-006: refresh token ở cookie, kiểm tra `Origin`.

## 7. Phân quyền hai lớp (authorization)

**Lớp 1, guard (theo role/permission):** `JwtAuthGuard` xác thực, `PermissionsGuard` đọc `@RequirePermissions('...')` rồi tra permission của các role từ cache (hết hạn sau 60 giây).

**Lớp 2, service (theo quyền sở hữu và trạng thái):** luôn gọi qua `CourseAccessService`, **không mỗi module tự viết kiểu riêng**. Định nghĩa: *"quản lý khoá"* = là giảng viên tạo khoá **hoặc** có permission `courses:manage-any` (ADMIN).

### 7.1 Danh mục permission Phase 1

| Permission | ADMIN | INSTRUCTOR | STUDENT |
|---|:-:|:-:|:-:|
| `users:read`, `users:create`, `users:update`, `users:lock`, `roles:manage` | ✓ | | |
| `courses:create` | ✓ | ✓ | |
| `courses:manage-any` (bỏ qua kiểm tra chủ khoá) | ✓ | | |
| `courses:read` (xem khoá đã công bố) | ✓ | ✓ | ✓ |
| `enrollments:request` | | | ✓ |
| `enrollments:manage` | ✓ | ✓ | |
| `progress:track` | | | ✓ |
| `progress:read-class` | ✓ | ✓ | |
| `assignments:manage`, `submissions:grade`, `grades:export` | ✓ | ✓ | |
| `assignments:submit` | | | ✓ |

Mã permission theo quy tắc `<resource>:<action>`; mỗi module tự thêm permission của mình bằng migration seed riêng (khớp `auth-spec.md`).

### 7.2 Quy tắc sở hữu và trạng thái (lớp 2)

| Tài nguyên | Hành động | Điều kiện | Khi vi phạm |
|---|---|---|---|
| Khoá | Xem danh sách đã công bố | Mọi user đã đăng nhập | |
| Khoá Nháp | Xem | Quản lý khoá | 404 (không tiết lộ tồn tại) |
| Khoá, chương, bài học | Sửa, đổi trạng thái, nhân bản | Quản lý khoá | 403 |
| Nội dung bài học | Đọc | Quản lý khoá, hoặc học viên APPROVED và khoá PUBLISHED/ENDED | 403 |
| Enrollment | Tạo yêu cầu | STUDENT, khoá PUBLISHED, chưa có bản ghi, không phải giảng viên của chính khoá đó | 409 / 403 |
| Enrollment | Duyệt, từ chối, thêm trực tiếp, xem danh sách lớp | Quản lý khoá | 403 |
| Enrollment | Xem của mình | Chính học viên | |
| Tiến độ | Ghi | Học viên APPROVED, khoá PUBLISHED | 403 |
| Tiến độ | Xem của mình / xem cả lớp | Chính học viên / quản lý khoá | 404 với tiến độ người khác |
| Bài tập | Tạo, sửa, xoá | Quản lý khoá | 403 |
| Bài tập | Xem | Quản lý khoá, hoặc học viên APPROVED | 403 |
| Bài nộp | Nộp, nộp lại | Chính học viên APPROVED, **trước hạn (giờ server)**, chưa được chấm, khoá PUBLISHED | 409 / 403 |
| Bài nộp | Xem | Chính học viên, hoặc quản lý khoá | **404** với bài của người khác |
| Điểm | Chấm, sửa điểm, xuất bảng điểm | Quản lý khoá | 403 |

**Nguyên tắc 403/404:** sai vai trò hoặc không phải chủ khoá → **403**; truy cập đối tượng riêng tư của người khác (bài nộp, tiến độ) hoặc khoá Nháp → **404**, để không lộ sự tồn tại.

## 8. Toàn vẹn dữ liệu và đồng thời

| Tình huống | Cách xử lý |
|---|---|
| Hai học viên cùng được duyệt vào chỗ cuối cùng | Duyệt trong transaction: khoá hàng khoá học (pessimistic write lock), đếm APPROVED, so với `max_students`, rồi cập nhật |
| Học viên bấm "Đăng ký" hai lần | `UNIQUE(course_id, student_id)`; bắt lỗi vi phạm ràng buộc, trả 409 `ENROLLMENT_DUPLICATE` |
| Nộp bài sát hạn | So sánh với **giờ server (UTC)**, không tin giờ client; upsert theo `UNIQUE(assignment_id, student_id)` |
| Nộp lại đúng lúc giảng viên chấm | Kiểm tra trạng thái chấm trong cùng transaction với việc ghi bài nộp |
| Sắp xếp lại chương/bài | Một transaction, đánh số lại `position` 1..n (không dùng UNIQUE trên `position`, xem `erd.md`) |
| Hai tab cùng gia hạn token | Frontend chỉ cho **một** lời gọi refresh đang chạy mỗi lúc (single-flight) và đồng bộ giữa các tab; chi tiết ở ADR-006 |

## 9. Hiệu năng: những chỗ dễ sinh N+1

| Màn hình | Nguy cơ | Giải pháp |
|---|---|---|
| Danh sách khoá (số chỗ còn lại, trạng thái đăng ký của tôi) | Mỗi khoá một truy vấn đếm | `getApprovedCounts`, `getEnrollmentStatuses` theo lô (GROUP BY / IN) |
| Tiến độ cả lớp | Mỗi học viên một truy vấn | Một truy vấn GROUP BY `lesson_progress` theo học viên, chia cho tổng số bài của khoá |
| Danh sách bài nộp | Mỗi học viên một truy vấn | LEFT JOIN từ enrollments APPROVED sang submissions |
| Xuất bảng điểm | Mỗi học viên × mỗi bài một truy vấn | Hai truy vấn (bài tập, bài nộp), ghép bằng map trong bộ nhớ |

Index khởi điểm (đầy đủ ở `erd.md`): `enrollments(course_id, status)`, `enrollments(student_id)`, `submissions(assignment_id)`, `lesson_progress(course_id, student_id)`, `courses(status)`, `courses(instructor_id, status)`, `assignments(course_id, due_at)`.

## 10. Các mối quan tâm xuyên suốt khác

| Chủ đề | Thiết kế |
|---|---|
| Validate | DTO + `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` |
| Lỗi | Exception filter chung trả `{ success:false, message, code, errors? }`; không lộ stack trace; chi tiết lỗi chỉ ghi log phía server |
| Response | Interceptor bọc `{ success, data, message }` (xem `api-conventions.md`) |
| Log | Log có cấu trúc (JSON), mỗi request một `requestId`; không ghi mật khẩu, token, cookie |
| Cấu hình | Biến môi trường, `.env.example`, kiểm tra bằng Zod khi khởi động (thiếu biến thì không chạy) |
| Thời gian | Lưu UTC (`DATETIME2`), truyền ISO-8601 UTC, hiển thị GMT+7 ở frontend |
| Rate limit | Chặt ở `/auth/login`, mức vừa phải toàn cục (ví dụ 100 yêu cầu/phút/IP) |
| CORS, HTTPS, header | Chỉ cho domain frontend; HTTPS bắt buộc ở production; dùng `helmet` |
| Markdown bài học | Hiển thị bằng trình render **không chạy HTML thô/script** (US-C05); giới hạn độ dài nội dung ở DTO |
| Sức khoẻ | `GET /health` kiểm tra kết nối database |

## 11. Test, môi trường, Git, CI, sao lưu

**Chiến lược test (Jest):** chạy trên **Node 24 trở lên**; script `npm test` gọi Jest kèm cờ `--experimental-vm-modules` vì các gói NestJS 12 phát hành dạng ESM (ADR-007).
- *Unit* cho service có logic lõi: auth (băm, xoay vòng token), duyệt đăng ký (sĩ số), tiến độ, tính điểm tổng
- *Integration không cần database*: đặt cạnh code (`*.spec.ts`), ví dụ lớp `common` (pipe, filter, interceptor, CORS, rate limit); chạy cùng `npm test`
- *E2E* (supertest, thư mục `test/`, database SQL Server thật): luồng chính và **ma trận phân quyền ở mục 7.2**. Dùng database `lms_test` (đọc `.env.test` khi `NODE_ENV=test`); `globalSetup` tự chạy migration trước khi test và **từ chối chạy nếu tên database không kết thúc bằng `_test`**
- *Migration:* mọi thay đổi schema đi qua migration. Phát triển dùng `npm run migration:*` (ts-node); triển khai chạy từ code đã build bằng `typeorm migration:run -d dist/database/data-source.js`; ứng dụng không tự chạy migration khi khởi động
- *Giao diện:* ít test, ưu tiên form phức tạp; quyết định công cụ khi làm

**Môi trường:**

| Môi trường | Mục đích | Ghi chú |
|---|---|---|
| dev | Máy bạn | SQL Server chạy bằng Docker; frontend dùng proxy tới backend để khỏi vướng CORS |
| test | CI và e2e | Database tạm, tạo từ migration |
| production | Chạy thật | Biến môi trường riêng; tuỳ chọn thêm *staging* nếu ngân sách cho phép |

SQL Server: bản Express miễn phí có giới hạn dung lượng mỗi database và tài nguyên; với quy mô này thường đủ, nhưng hãy kiểm tra giới hạn của phiên bản hiện hành trước khi chọn hosting (ở giai đoạn 7).

**Git và CI:** nhánh `feature/*`, `fix/*` vào `develop`, rồi `main`; Conventional Commits. CI chạy lint, type-check, unit test, e2e, build mỗi lần push.

**Sao lưu:** hằng ngày, giữ tối thiểu 7 bản gần nhất, lưu ngoài máy chủ chạy database; **thử khôi phục** ít nhất một lần trước khi chạy thật.

## 12. Thư viện dự kiến (cần bạn xác nhận)

Theo quy ước project, thư viện ngoài stack nêu lý do; lựa chọn lớn có ADR kèm so sánh.

| Thư viện | Dùng để | Ghi chú |
|---|---|---|
| `@nestjs/typeorm`, `typeorm`, `mssql` | Truy cập SQL Server, migration | **[ĐÃ CHỐT]** ADR-004 so sánh với Prisma |
| `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt` | JWT | Cách chuẩn trong NestJS |
| `bcrypt` | Băm mật khẩu | Theo quy ước project |
| `class-validator`, `class-transformer` | DTO validate | Theo quy ước project |
| `@nestjs/throttler` | Rate limit | Tích hợp sẵn với NestJS |
| `@nestjs/config`, `zod` | Cấu hình, kiểm tra biến môi trường | Dùng chung Zod với frontend, đỡ học thêm thư viện |
| `helmet`, `cookie-parser` | Header bảo mật; đọc cookie (nếu ADR-006 được chọn) | |
| `jest`, `supertest` | Test | Mặc định của NestJS; project CommonJS (ADR-007) |
| `oxlint`, `prettier` | Lint, định dạng code | Mặc định của Nest CLI 12 (thay ESLint) |
| `react-router` | Điều hướng | Theo quy ước project |
| `@tanstack/react-query` | Data fetching, cache | **[ĐÃ CHỐT]** ADR-005 |
| `react-hook-form`, `zod`, `@hookform/resolvers` | Form và validate | Theo quy ước project |
| `react-markdown` | Hiển thị Markdown an toàn | Mặc định không chạy HTML thô; cần cho US-C05 |
| HTTP client | Gọi API | Dùng `fetch` bọc sẵn (không thêm thư viện); lớp bọc xử lý token và refresh một chỗ |

## 13. Điểm mở rộng cho Phase 2 (thiết kế đã chừa chỗ)

| Hạng mục | Cách mở rộng không phá schema hiện có |
|---|---|
| Đính kèm tệp, nộp bài bằng tệp | Thêm bảng `lesson_attachments`, `submission_files` và module `storage` |
| Thông báo (US-S11) | Thêm bảng `notifications`; phát sự kiện từ `enrollments`/`assignments` |
| Gỡ học viên (US-E06) | Thêm giá trị trạng thái `REMOVED` (cột trạng thái dùng VARCHAR + CHECK nên dễ thêm) |
| Khoá mẫu và lớp | Thêm bảng `course_templates`, cột `template_id` nullable ở `courses` (ADR-002) |
| Trọng số điểm (US-S10) | Thêm cột `weight` ở `assignments`; điểm tổng đã được tính khi cần nên không phải migrate dữ liệu |
| Nộp trễ (US-S12) | Thêm cột cấu hình ở `assignments` |
| Điểm danh | Thêm bảng `sessions`, `attendances` |

## 14. Kiểm chứng thiết kế: đi bộ qua luồng cốt lõi

| # | Bước (PRD mục 5.1) | Màn hình | Endpoint | Bảng | Quyền / điều kiện |
|---|---|---|---|---|---|
| 1 | ADMIN tạo tài khoản giảng viên | `/admin/users` | `POST /users` | users, user_roles | `users:create` |
| 2 | Giảng viên dựng khoá, công bố | `/manage/courses/:id` | `POST /courses`, chương, bài học, `PUT .../outline`, `PATCH .../status` | courses, chapters, lessons | Quản lý khoá |
| 3 | Học viên đăng ký, gửi yêu cầu | `/register`, `/courses/:id` | `POST /auth/register`, `POST /courses/:id/enrollments` | users, enrollments | `enrollments:request`, khoá PUBLISHED |
| 4 | Giảng viên duyệt | tab Học viên | `PATCH /enrollments/:id/status` | enrollments | Quản lý khoá, còn chỗ (transaction) |
| 5 | Học và nộp bài | `/learn/...` | `GET /lessons/:id`, `PUT /lessons/:id/progress`, `PUT /assignments/:id/submission` | lesson_progress, submissions | Học viên APPROVED, trước hạn |
| 6 | Chấm điểm | tab Bài tập | `GET /assignments/:id/submissions`, `PUT /submissions/:id/grade` | submissions | Quản lý khoá |
| 7 | Xuất bảng điểm | tab Điểm | `GET /courses/:id/grades/export` | assignments, submissions, enrollments | Quản lý khoá |

**Lỗ hổng phát hiện khi đi bộ và cách xử lý:**

| # | Phát hiện | Xử lý |
|---|---|---|
| a | `GET /courses/:id` cho học viên phải trả thêm **trạng thái đăng ký của chính họ** để nút "Đăng ký/Chờ duyệt/Vào học" hiển thị đúng (US-E01, E04) | Đã đưa vào thiết kế (`myEnrollment`) qua `getEnrollmentStatuses` |
| b | Danh sách khoá cần "số chỗ còn lại" | `getApprovedCounts` theo lô |
| c | Công bố → Nháp chỉ được khi chưa có học viên APPROVED (US-C03) | `CourseAccessService` cung cấp đếm APPROVED |
| d | Học viên bị **từ chối** rồi muốn đăng ký lại: `UNIQUE(course, student)` chặn tạo bản ghi mới, story chưa nói rõ | **Quy tắc đề xuất:** giảng viên có thể duyệt lại bản ghi REJECTED; học viên không tự đăng ký lại. Cần bạn xác nhận (mục 15) |
| e | Giảng viên **thêm trực tiếp** (US-E03) một học viên đang PENDING/REJECTED | Chuyển bản ghi hiện có sang APPROVED thay vì báo trùng |
| f | Điểm tổng (US-S09): "bài đã giao, bài chưa nộp tính 0" chưa rõ với bài **chưa đến hạn** | Đề xuất: bài chưa đến hạn và chưa có điểm **không tính** (tránh kéo điểm xuống oan); cập nhật AC của US-S09 ở spec module |
| g | Giảng viên cũng có thể là học viên (nhiều role) | Chặn đăng ký vào chính khoá do mình dạy |
| h | Cần có tài khoản ADMIN đầu tiên | Seed từ `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD` (đã có trong `auth-spec.md`) |

## 15. Quyết định thiết kế (đã xác nhận 09/10/2026)

| # | Quyết định | Kết quả | Tài liệu |
|---|---|---|---|
| 1 | Chấp nhận hoãn US-C06, US-S04, US-E06 | Có | `02-user-stories.md` mục 4 |
| 2 | Một khoá = một đợt | Có | ADR-002 |
| 3 | Không lưu % tiến độ và điểm tổng | Có | ADR-003 |
| 4 | ORM | TypeORM | ADR-004 |
| 5 | Data fetching ở frontend | TanStack Query | ADR-005 |
| 6 | Refresh token ở cookie HttpOnly | Có | ADR-006 (`auth-spec.md` đã cập nhật v1.1) |
| 7 | Quy tắc duyệt lại học viên bị từ chối | Giảng viên duyệt lại được | mục 14 (d) |
| 8 | Bài chưa đến hạn không tính vào điểm tổng | Có | mục 14 (f) |

## 16. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 0.1 | 09/10/2026 | Bản nháp đầu tiên |
| 0.2 | 09/10/2026 | Chốt 8 quyết định ở mục 15; ADR-001 đến 006 chuyển sang Đã chấp nhận |
| 0.3 | 10/10/2026 | Ghi nhận Node 24, Jest chạy với cờ ESM, oxlint (ADR-007); chốt NestJS 12 dạng CommonJS |
| 0.4 | 10/10/2026 | Chi tiết chiến lược test (integration không DB, e2e với `lms_test`) và quy trình migration |
