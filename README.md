# Quy trình xây dựng một ứng dụng từ zero

Có thể hình dung gồm hai phần: **chuẩn bị một lần** (giai đoạn 0 đến 4), rồi **lặp lại cho từng module** (giai đoạn 5), cuối cùng là phát hành và vận hành.

```
0. Ý tưởng ─► 1. PRD ─► 2. User Story ─► 3. Thiết kế ─► 4. Setup
                                                            │
        ┌───────────────────────────────────────────────────┘
        ▼
   5. Xây từng module (lặp)  ──►  6. Kiểm thử  ──►  7. Triển khai  ──►  8. Vận hành
   spec → DB → API → UI → test        ▲                                   │
        ▲                             └───────── feedback ◄───────────────┘
        └─ module kế tiếp
```

## Giai đoạn 0: Xác định vấn đề
- **Việc làm:** trả lời ai dùng, họ đang gặp khó gì, thành công trông như thế nào.
- **Đầu ra:** 3-5 câu mô tả ý tưởng và đối tượng người dùng.

## Giai đoạn 1: PRD dự án
- **Việc làm:** viết 1-2 trang gồm mục tiêu, các role, **MVP gồm gì và không gồm gì**, yêu cầu phi chức năng.
- **Đầu ra:** `docs/prd.md`.
- **Lưu ý:** phần "ngoài scope" quan trọng không kém phần "trong scope", vì nó chặn việc làm lan man.

## Giai đoạn 2: User Story và ưu tiên
- **Việc làm:** liệt kê story theo role, gắn AC, xếp ưu tiên (Must/Should/Could), nhóm thành các module và chia phase.
- **Đầu ra:** `docs/user-stories.md` kèm thứ tự module.
- **Ví dụ LMS:** auth, courses, lessons, enrollments, progress, rồi mới đến đánh giá, thông báo.

## Giai đoạn 3: Thiết kế (vừa đủ, không thiết kế hết trước)
- **Kiến trúc tổng thể:** các thành phần và cách chúng nói chuyện với nhau (đã chốt: React, NestJS, SQL Server, JWT).
- **Sơ đồ DB (ERD) mức tổng quan:** các thực thể chính và quan hệ. Chi tiết từng bảng để lúc làm module đó.
- **Quy ước:** đặt tên, format response, xử lý lỗi, cấu trúc thư mục, quy ước Git.
- **Wireframe:** phác các màn hình chính (giấy hoặc Figma), chưa cần đẹp.
- **Đầu ra:** `docs/architecture.md`, `docs/erd.md`, wireframe.

## Giai đoạn 4: Khởi tạo dự án (Setup)
- **Việc làm:** tạo repo Git, nhánh `main` và `develop`, khởi tạo NestJS và React, cấu hình ESLint/Prettier, `.env.example`, kết nối database, ValidationPipe, exception filter, response interceptor.
- **Đầu ra:** project chạy được, "Hello World" gọi từ frontend sang backend và chạm tới database.
- **Mẹo:** nếu có thể, thiết lập thêm CI đơn giản (chạy lint và test mỗi lần push) ngay từ đầu, càng muộn càng khó thêm.

## Giai đoạn 5: Xây từng module theo lát cắt dọc (vertical slice)
Mỗi module đi trọn một vòng, **không làm toàn bộ backend rồi mới làm toàn bộ frontend**:

1. Viết spec module (bảng DB, API, quyền)
2. Migration và entity
3. DTO, service, controller, guard
4. Viết test cho logic quan trọng
5. Service và trang frontend
6. Kiểm tra cả luồng, commit, merge

Mỗi bước nhỏ nên chạy được và commit riêng (Conventional Commits). Làm xong một module mới sang module tiếp theo, nhờ đó luôn có một sản phẩm dùng được dù mới nhỏ.

## Giai đoạn 6: Kiểm thử
- **Unit test (Jest):** logic nghiệp vụ như auth, đăng ký học, tính điểm.
- **Integration/e2e:** gọi API thật với database thử nghiệm.
- **Kiểm thử thủ công theo AC:** đi qua từng story như người dùng thật.
- **Rà soát bảo mật:** phân quyền, rò rỉ dữ liệu, rate limit, lộ thông tin trong response.

## Giai đoạn 7: Triển khai
- **Môi trường:** tách `dev`, `staging`, `production`; cấu hình bí mật qua biến môi trường.
- **Việc làm:** build, chạy migration trên môi trường thật, cấu hình HTTPS và CORS, thiết lập **sao lưu database**.
- **CI/CD:** tự động test, build, deploy khi merge vào nhánh chính.
- **Đầu ra:** bản chạy thật, có cách quay lại (rollback) khi lỗi.

## Giai đoạn 8: Vận hành và cải tiến
- **Theo dõi:** log có cấu trúc, giám sát lỗi, uptime.
- **Thu thập feedback**, đưa vào backlog, xếp ưu tiên rồi quay lại giai đoạn 2 cho phase tiếp theo.

## Năm nguyên tắc nên giữ
1. **MVP trước:** làm ít nhưng chạy trọn từ đăng nhập đến dùng được tính năng chính.
2. **Thiết kế vừa đủ cho module đang làm**, đừng cố thiết kế cả hệ thống chi tiết từ đầu.
3. **Mỗi lần một bước nhỏ, chạy được.**
4. **Tài liệu ngắn nhưng luôn đồng bộ với code**; code mâu thuẫn tài liệu thì cập nhật ngay.
5. **Làm xong rồi mới tối ưu:** đúng trước, sạch sau, nhanh sau cùng.

## Bạn đang ở đâu trong quy trình này?

Dự án LMS của bạn đã có tài liệu module auth (đoạn giữa giai đoạn 2 và 3). Phần còn thiếu so với quy trình là:
- **PRD chung của cả dự án** (giai đoạn 1) và danh sách module kèm thứ tự (giai đoạn 2)
- **ERD tổng quan** (giai đoạn 3)
- **Khởi tạo project** (giai đoạn 4)

**Bước tiếp theo đề xuất:** soạn PRD dự án LMS 1-2 trang và danh sách module theo thứ tự ưu tiên, vì đó là nền để chốt phạm vi MVP. Bạn đã có PRD chung chưa? Nếu chưa, mình sẽ hỏi bạn tối đa 3 câu về phạm vi rồi soạn luôn.