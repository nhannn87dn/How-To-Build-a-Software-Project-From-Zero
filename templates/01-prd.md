# PRD: Hệ thống quản lý khoá học trực tuyến (LMS)

> Phiên bản: 0.2 (Draft) | Ngày: 09/10/2026 | Người viết: Product owner kiêm developer
> Nguồn: `00-idea.md` (tình huống giả định bên dưới)
> **Quy ước:** mục đánh dấu **[GIẢ ĐỊNH]** là số liệu/quyết định tự đặt cho tình huống giả định, cần thay bằng dữ liệu thật hoặc kiểm chứng trước khi chốt v1.0.

---

## 1. Bối cảnh & vấn đề

**Tình huống:** Một giảng viên lập trình độc lập (kèm 1-2 trợ giảng) dạy theo mô hình lớp nhỏ, mở khoá theo đợt. Mỗi đợt có 20-40 học viên, mỗi khoá học khoảng 12-16 buổi, học 2-3 buổi/tuần, thường chạy song song 3-5 khoá. **[GIẢ ĐỊNH]**

**Hiện trạng đang vận hành:**

| Việc | Công cụ hiện tại |
|---|---|
| Nhận đăng ký, chốt danh sách | Google Form + Excel |
| Trao đổi, thông báo, nộp bài | Nhóm Zalo, email |
| Tài liệu, slide, video ghi hình | Google Drive |
| Điểm danh, điểm số, nhận xét | Mỗi khoá một file Google Sheet |
| Thu học phí | Chuyển khoản ngân hàng, đối chiếu tay |

**Vấn đề (Problem Statement):** Giảng viên và trợ giảng gặp khó khăn trong việc theo dõi bài nộp, tiến độ và điểm của từng học viên vì dữ liệu nằm rải rác ở 4-5 nơi, dẫn đến mất nhiều giờ mỗi tuần cho việc quản lý thủ công, bài nộp bị sót, học viên không biết mình đang học đến đâu và dễ bỏ giữa chừng. Mỗi khoá mới lại phải dựng lại cấu trúc từ đầu.

**Pain point (số liệu ước lượng, [GIẢ ĐỊNH]):**

| # | Pain point | Tần suất | Tốn | Ai chịu |
|---|---|---|---|---|
| P1 | Tìm bài nộp của một học viên (lục Zalo/email) | Nhiều lần/tuần | ~10 phút/lần | Giảng viên, trợ giảng |
| P2 | Tổng hợp điểm cuối khoá từ nhiều Sheet | Mỗi khoá | ~2 giờ | Giảng viên |
| P3 | Dựng khoá mới (copy cấu trúc, tài liệu, danh sách) | Mỗi đợt | ~3-4 giờ | Giảng viên |
| P4 | Học viên hỏi "em đã nộp bài chưa?", "em còn thiếu bài nào?" | Hằng ngày | ~30 phút/ngày | Giảng viên, học viên |
| P5 | Không phát hiện sớm học viên bỏ dở | Liên tục | Tỷ lệ bỏ học cao | Giảng viên, học viên |
| P6 | Cấp lại tài khoản/quyền truy cập tài liệu | Vài lần/khoá | ~15 phút/lần | Giảng viên |

## 2. Mục tiêu & chỉ số thành công

| Mục tiêu | Gắn pain point | Chỉ số | Hiện tại (ước lượng) | Mục tiêu |
|---|---|---|---|---|
| G1: Mọi bài nộp ở một nơi, truy cập nhanh | P1 | Thời gian tìm bài nộp của 1 học viên | ~10 phút | Dưới 1 phút |
| G2: Điểm tự tổng hợp, xuất được | P2 | Thời gian có bảng điểm cuối khoá | ~2 giờ | Dưới 15 phút |
| G3: Dựng khoá mới nhanh bằng nhân bản | P3 | Thời gian dựng khoá mới | ~3-4 giờ | Dưới 45 phút |
| G4: Học viên tự xem tiến độ và bài còn thiếu | P4, P5 | Số câu hỏi "đã nộp chưa" qua Zalo/ngày; % học viên xem tiến độ mỗi tuần | ~10 câu/ngày | Dưới 2 câu/ngày; trên 70% học viên xem tiến độ hằng tuần |
| G5: Quản lý tài khoản không cần can thiệp database | P6 | Thời gian cấp/khoá/khôi phục quyền | ~15 phút | Dưới 3 phút |

**Tiêu chí thành công tổng thể (sau 1 đợt chạy thật):** giảng viên dùng hệ thống làm nơi quản lý chính cho ít nhất 1 khoá, và hết đợt không phải quay lại Excel/Sheet để tổng hợp điểm.

## 3. Non-goals (không làm trong Phase 1)

- Thanh toán online và hoá đơn (học phí vẫn thu ngoài hệ thống)
- Ứng dụng mobile native (chỉ web responsive)
- Livestream, phòng học trực tuyến (vẫn dùng Google Meet/Zoom bên ngoài)
- Tự lưu trữ và phát video (dùng link nhúng từ YouTube unlisted/Drive)
- Tải tệp lên hệ thống: đính kèm tài liệu vào bài học và nộp bài bằng tệp (Phase 1 dùng liên kết Drive/GitHub, tệp để Phase 2)
- Gỡ học viên khỏi khoá (Phase 2; Phase 1 chỉ xem danh sách và từ chối/duyệt)
- Chấm bài tự động bằng chạy code, quiz trắc nghiệm có chấm điểm tự động
- Diễn đàn, chat nội bộ (vẫn dùng Zalo)
- Chứng chỉ có chữ ký số, đa ngôn ngữ, đa trung tâm (multi-tenant)
- Đăng nhập Google/Facebook, quên mật khẩu qua email (xem Phase 2)

## 4. Người dùng & vai trò

| Role | Mô tả | Số lượng dự kiến [GIẢ ĐỊNH] |
|---|---|---|
| ADMIN | Quản trị hệ thống và tài khoản (thường là giảng viên chính hoặc người vận hành) | 1-2 |
| INSTRUCTOR | Giảng viên/trợ giảng: dựng khoá, đăng bài học, giao và chấm bài | 2-5 |
| STUDENT | Học viên: đăng ký, học, nộp bài, xem điểm và tiến độ | 100-200 tài khoản, 30-60 hoạt động cùng lúc |

**Ma trận quyền sơ bộ:**

| Chức năng | ADMIN | INSTRUCTOR | STUDENT |
|---|:-:|:-:|:-:|
| Quản lý tài khoản, gán role | ✓ | | |
| Tạo/sửa khoá học của mình | ✓ | ✓ | |
| Sửa khoá học của giảng viên khác | ✓ | | |
| Xem danh sách khoá đã công bố | ✓ | ✓ | ✓ |
| Duyệt đăng ký học | ✓ | ✓ (khoá của mình) | |
| Học bài, nộp bài | | | ✓ |
| Chấm bài, nhận xét | ✓ | ✓ (khoá của mình) | |
| Xem điểm/tiến độ của chính mình | | | ✓ |
| Xem điểm/tiến độ cả lớp | ✓ | ✓ (khoá của mình) | |
| Xem dữ liệu của học viên khác | ✓ | ✓ (khoá của mình) | ✗ |

## 5. Phạm vi

### 5.1 Luồng cốt lõi (MVP phải chạy trọn)

1. ADMIN tạo tài khoản giảng viên.
2. Giảng viên tạo khoá học, thêm chương/bài học và bài tập, công bố khoá.
3. Học viên đăng ký tài khoản, gửi yêu cầu đăng ký khoá.
4. Giảng viên/ADMIN duyệt (sau khi nhận học phí ngoài hệ thống).
5. Học viên vào học, đánh dấu hoàn thành bài, nộp bài tập.
6. Giảng viên chấm điểm, nhận xét; học viên xem kết quả và tiến độ.
7. Cuối khoá giảng viên xuất bảng điểm.

> Quy tắc cắt phạm vi: bỏ năng lực nào mà luồng trên vẫn chạy được thì năng lực đó không phải Must.

### 5.2 Phân loại MoSCoW

| Năng lực | MoSCoW | Phase |
|---|---|---|
| Tài khoản, xác thực, phân quyền | Must | 1 |
| Quản lý khoá học, chương, bài học | Must | 1 |
| Đăng ký học và duyệt | Must | 1 |
| Học bài và theo dõi tiến độ | Must | 1 |
| Giao bài, nộp bài, chấm điểm | Must | 1 |
| Xuất bảng điểm (CSV mở được trong Excel; .xlsx để sau) | Must | 1 |
| Nhân bản khoá học | Should | 1 (cuối) |
| Thông báo trong hệ thống (bài mới, đã chấm) | Should | 1 hoặc 2 |
| Điểm danh theo buổi | Should | 2 |
| Dashboard thống kê, cảnh báo học viên bỏ dở | Could | 2 |
| Quên mật khẩu qua email, xác thực email | Could | 2 |
| Đính kèm tài liệu vào bài học, nộp bài bằng tệp | Should | 2 |
| Gỡ học viên khỏi khoá | Should | 2 |
| Thanh toán online | Won't | Sau |

### 5.3 Kiểm tra sức chứa (ước lượng thô, [GIẢ ĐỊNH])

Thời gian có thể dành: khoảng 15 giờ/tuần × 12 tuần = 180 giờ, chừa 30% cho lỗi và thay đổi, còn khoảng 125 giờ cho cả khởi tạo dự án và tính năng.

Ước lượng dưới đây lấy từ `02-user-stories.md` (1 điểm ≈ 1,5 giờ) **sau khi chuyển 3 hạng mục sang Phase 2** (đính kèm tài liệu, nộp bài bằng tệp, gỡ học viên):

| Hạng mục | Ước lượng (giờ) |
|---|---|
| Setup dự án, CI cơ bản, deploy | 20-25 |
| Auth, users, roles, permissions | ~24 |
| Khoá học, chương, bài học | ~27 |
| Đăng ký học và duyệt | ~17 |
| Tiến độ học | ~12 |
| Bài tập, nộp bài (liên kết), chấm điểm, xuất điểm | ~28 |
| **Tổng Must** | **~130** |

Nhận xét: Must vẫn **sát giới hạn** (vượt khoảng 5 giờ so với quỹ ~125 giờ), nên Should (nhân bản khoá, thông báo) chỉ làm nếu còn thời gian, nếu không chuyển sang Phase 2. Đo lại tốc độ thực tế sau đợt đầu tiên rồi điều chỉnh.

## 6. Yêu cầu chức năng

Mã dạng `FR-<NHÓM>-<số>`; mỗi FR sẽ được chia thành User Story ở giai đoạn sau.

### Tài khoản & phân quyền (AUTH)
| Mã | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-AUTH-01 | Khách tự đăng ký tài khoản học viên | Must |
| FR-AUTH-02 | Đăng nhập, đăng xuất, giữ phiên làm việc | Must |
| FR-AUTH-03 | ADMIN tạo tài khoản giảng viên và quản lý user (tìm, khoá/mở khoá) | Must |
| FR-AUTH-04 | ADMIN gán role cho user | Must |
| FR-AUTH-05 | Người dùng xem/sửa hồ sơ, đổi mật khẩu | Should |
| FR-AUTH-06 | ADMIN chỉnh permission của role | Could |

### Khoá học (COURSE)
| Mã | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-COURSE-01 | Giảng viên tạo/sửa khoá học (tên, mô tả, thời gian, sĩ số tối đa, trạng thái nháp/công bố/đã kết thúc) | Must |
| FR-COURSE-02 | Giảng viên tổ chức khoá thành chương và bài học (thứ tự sắp xếp được) | Must |
| FR-COURSE-03 | Bài học gồm nội dung văn bản và link video nhúng (tài liệu chia sẻ dưới dạng liên kết trong nội dung) | Must |
| FR-COURSE-04 | Học viên xem danh sách khoá đã công bố, tìm kiếm | Must |
| FR-COURSE-05 | Giảng viên nhân bản khoá học (giữ cấu trúc và nội dung, bỏ học viên và điểm) | Should |
| FR-COURSE-06 | Giảng viên đính kèm tệp tài liệu vào bài học | Should (Phase 2) |

### Đăng ký học (ENROLL)
| Mã | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-ENROLL-01 | Học viên gửi yêu cầu đăng ký khoá đã công bố; không đăng ký trùng | Must |
| FR-ENROLL-02 | Giảng viên/ADMIN duyệt hoặc từ chối (yêu cầu đã từ chối có thể được duyệt lại); có thể thêm học viên trực tiếp vào khoá | Must |
| FR-ENROLL-03 | Học viên xem danh sách khoá của mình và trạng thái đăng ký | Must |
| FR-ENROLL-04 | Giảng viên xem danh sách học viên của khoá | Must |
| FR-ENROLL-05 | Giảng viên gỡ học viên khỏi khoá (giữ nguyên bài nộp và điểm cũ) | Should (Phase 2) |

### Học tập & tiến độ (PROGRESS)
| Mã | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-PROGRESS-01 | Học viên đánh dấu hoàn thành từng bài học | Must |
| FR-PROGRESS-02 | Học viên xem % tiến độ và bài còn thiếu của mình | Must |
| FR-PROGRESS-03 | Giảng viên xem tiến độ cả lớp theo từng học viên | Must |

### Bài tập & điểm (ASSIGN)
| Mã | Yêu cầu | Ưu tiên |
|---|---|---|
| FR-ASSIGN-01 | Giảng viên tạo bài tập (đề, hạn nộp, thang điểm) gắn với khoá/bài học | Must |
| FR-ASSIGN-02 | Học viên nộp bài bằng liên kết (repo) và ghi chú trước hạn; được nộp lại đến hết hạn | Must |
| FR-ASSIGN-03 | Giảng viên xem toàn bộ bài nộp của một bài tập, lọc đã nộp/chưa nộp | Must |
| FR-ASSIGN-04 | Giảng viên chấm điểm và nhận xét; học viên xem kết quả | Must |
| FR-ASSIGN-05 | Xuất bảng điểm của khoá ra CSV (mở được trong Excel); định dạng .xlsx để sau | Must |
| FR-ASSIGN-06 | Thông báo trong hệ thống khi có bài mới/đã chấm | Should |
| FR-ASSIGN-07 | Học viên nộp bài bằng tệp (pdf, zip, ...) | Should (Phase 2) |

## 7. Yêu cầu phi chức năng (đo được)

| Nhóm | Yêu cầu |
|---|---|
| Hiệu năng | Trang danh sách khoá và danh sách bài nộp phản hồi dưới 2 giây với 1.000 bản ghi (phân trang); đăng nhập dưới 500 ms (p95) |
| Quy mô | Chịu được 100 người dùng đồng thời mà không suy giảm đáng kể |
| Bảo mật | Mật khẩu băm bcrypt; JWT access token ngắn hạn kèm refresh token (lưu ở cookie HttpOnly, ADR-006); phân quyền kiểm tra ở server; học viên không truy cập được dữ liệu của học viên khác; rate limit đăng nhập; không lộ stack trace |
| Liên kết và nội dung nhập | Phase 1 **không nhận tệp tải lên**; chỉ nhận liên kết `https` (kiểm tra định dạng, `videoUrl` chỉ nhận YouTube/Google Drive); Markdown hiển thị không chạy script. Khi làm tệp ở Phase 2: giới hạn 10 MB/tệp, chỉ nhận định dạng cho phép (pdf, zip, png, jpg, md, txt), kiểm tra loại tệp ở server, không thực thi tệp |
| Khả dụng | Giao diện responsive, dùng tốt trên điện thoại (màn hình từ 360 px) |
| Thời gian & định dạng | Lưu thời gian dạng UTC, hiển thị theo GMT+7; ngày định dạng dd/mm/yyyy; hạn nộp bài tính theo GMT+7 |
| Dữ liệu | Sao lưu database tự động hằng ngày, giữ tối thiểu 7 bản gần nhất; thử khôi phục ít nhất 1 lần trước khi chạy thật |
| Bảo trì | Thay đổi schema qua migration; có test cho auth, đăng ký học, tính tiến độ, tính điểm; CI chạy lint và test mỗi lần push |
| Quan sát | Log có cấu trúc, không ghi mật khẩu/token; có endpoint kiểm tra sức khoẻ |

## 8. Ràng buộc, giả định, rủi ro, phụ thuộc

**Ràng buộc:** tech stack cố định (React + TypeScript, NestJS + TypeScript, SQL Server, JWT); một người phát triển; ngân sách hạ tầng thấp (một máy chủ nhỏ).

**Giả định và cách kiểm chứng:**

| Giả định | Cách kiểm chứng |
|---|---|
| Học viên chịu dùng web thay cho Zalo để nộp bài và xem điểm | Hỏi 5 học viên đợt gần nhất; chạy thử 1 khoá nhỏ |
| Học phí thu ngoài hệ thống, duyệt thủ công là chấp nhận được | Xác nhận với giảng viên chính về quy trình thu tiền |
| Link video nhúng (YouTube unlisted/Drive) đủ dùng, không cần tự lưu video | Kiểm tra số bài có video và nơi đang lưu |
| Một khoá tương ứng một đợt học (chưa tách "khoá mẫu" và "lớp") | **Đã chốt** ngày 09/10/2026 (ADR-002); xem lại khi cần chạy nhiều đợt song song từ cùng nội dung |
| Nộp bài bằng liên kết repo và ghi chú là đủ cho Phase 1 (chưa cần nộp tệp) | Rà soát các bài tập đợt gần nhất; có bài nào bắt buộc nộp tệp không |

**Rủi ro:**

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Phình scope (thêm tính năng giữa chừng) | Cao | Giữ mục Non-goals; mọi yêu cầu mới ghi vào backlog, xét ở đầu phase sau |
| Làm một mình, ít review, dễ nợ kỹ thuật | Trung bình | Test cho logic cốt lõi, commit nhỏ, tự review trước khi merge |
| Học viên không chuyển sang dùng hệ thống | Cao | Chạy thử một khoá nhỏ trước; giảng viên chỉ nhận bài qua hệ thống |
| Mất dữ liệu | Cao | Sao lưu hằng ngày, thử khôi phục |
| Lộ bài nộp/điểm giữa học viên | Cao | Kiểm tra quyền ở cả guard và service; test phân quyền |
| Tải tệp độc hại | Thấp ở Phase 1 | Phase 1 không nhận tệp; khi làm Phase 2 áp dụng giới hạn loại/kích thước, không thực thi, lưu ngoài thư mục web |
| XSS qua nội dung người dùng nhập (Markdown, ghi chú, nhận xét) | Trung bình | Render không chạy HTML/script; refresh token ở cookie HttpOnly (ADR-006) |

**Phụ thuộc giữa các module:** AUTH → COURSE → ENROLL → PROGRESS → ASSIGN (ASSIGN cũng cần ENROLL để biết học viên nào thuộc khoá).

## 9. Roadmap theo phase

| Phase | Nội dung | Điều kiện hoàn thành |
|---|---|---|
| **1 (MVP)** | AUTH, COURSE, ENROLL, PROGRESS, ASSIGN (Must); Should nếu còn thời gian | Chạy thật được một khoá với học viên thật từ đăng ký đến xuất điểm |
| **2** | Đính kèm tài liệu và nộp bài bằng tệp, gỡ học viên khỏi khoá, điểm danh theo buổi, dashboard và cảnh báo học viên bỏ dở, quên mật khẩu qua email, thông báo qua email | Giảng viên không còn dùng Sheet cho điểm danh; có cảnh báo sớm |
| **3** | Thanh toán/đối soát học phí, chứng nhận hoàn thành, tách "khoá mẫu" và "lớp", quiz | Theo nhu cầu thực tế sau Phase 2 |

## 10. Câu hỏi còn mở và quyết định

| # | Câu hỏi | Trạng thái | Ảnh hưởng |
|---|---|---|---|
| 1 | Một khoá có mở nhiều đợt/lớp không? | **Đã chốt 09/10/2026:** Phase 1 một khoá = một đợt, mở đợt mới bằng nhân bản (ADR-002) | DB, nhân bản khoá |
| 2 | Học viên có tự động được duyệt với khoá miễn phí không? | Chưa chốt; giả định mặc định: mọi đăng ký đều qua duyệt (US-E07, Could) | Luồng ENROLL |
| 3 | Trợ giảng có quyền chấm bài ở mọi khoá hay chỉ khoá được gán? | Chưa chốt; giả định mặc định: chỉ giảng viên tạo khoá và ADMIN. Nếu muốn gán trợ giảng cho khoá, cần bổ sung FR | Phân quyền |
| 4 | Điểm cuối khoá tính thế nào? | **Đã chốt cho Phase 1 (ADR-003):** trung bình cộng quy về thang 10, chỉ tính bài đã đến hạn hoặc đã có điểm, bài đã đến hạn chưa nộp tính 0. Trọng số và điều kiện đạt để Phase 2 (US-S10) | FR-ASSIGN-05, test tính điểm |
| 5 | Có cho học viên nộp trễ (trừ điểm) hay khoá cứng sau hạn? | Chưa chốt; giả định mặc định: khoá cứng sau hạn (US-S12, Could) | Luồng nộp bài |
| 6 | Một user có nhiều role hay một role? | Đang dùng giả định: nhiều role (thiết kế auth đã theo hướng này) | DB module auth |
| 7 | Có cần xoá user thật, hay chỉ khoá? | Đang dùng giả định: chỉ khoá | DB, quy định dữ liệu |

**Quyết định bổ sung chốt ngày 09/10/2026 (từ giai đoạn thiết kế):**
- Học viên bị từ chối không tự đăng ký lại; giảng viên có thể duyệt lại yêu cầu đã từ chối
- Giảng viên không đăng ký vào khoá do chính mình dạy
- Hoãn sang Phase 2: đính kèm tệp (FR-COURSE-06), nộp bài bằng tệp (FR-ASSIGN-07), gỡ học viên (FR-ENROLL-05)

## 11. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 0.1 | 09/10/2026 | Bản nháp đầu tiên theo tình huống giả định |
| 0.2 | 09/10/2026 | Chuyển FR-COURSE-06, FR-ASSIGN-07, FR-ENROLL-05 (tách từ FR-COURSE-03, FR-ASSIGN-02, FR-ENROLL-04) sang Phase 2; chốt câu hỏi #1 và #4 cho Phase 1; thêm quy tắc duyệt lại học viên bị từ chối; xuất bảng điểm Phase 1 là CSV; cập nhật sức chứa, NFR, rủi ro, roadmap |
