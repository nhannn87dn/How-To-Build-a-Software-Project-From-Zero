# User Stories: Hệ thống quản lý khoá học trực tuyến (LMS)

> Phiên bản: 0.1 (Draft) | Ngày: 09/10/2026 | Nguồn: `01-prd.md` (v0.1)
> Module auth đã có tài liệu riêng (`auth.md`); mã story E1 dưới đây **giữ nguyên** US-A01 đến US-A10 để khớp.

---

## 1. Quy ước

**Mã story:** `US-<epic><số>`: A = Auth, C = Course, E = Enrollment, P = Progress, S = assignment & Score.

**Ưu tiên:** kế thừa MoSCoW từ PRD. Story nào mình đề xuất đổi ưu tiên được gắn **[ĐỀ XUẤT HOÃN]** và chưa tự ý đổi; bạn cần xác nhận rồi mới cập nhật PRD.

**Nhãn trạng thái:**
- **[CHỜ QUYẾT ĐỊNH #n]**: story phụ thuộc câu hỏi mở số n trong PRD; dùng **giả định mặc định** ghi trong mục 6 để vẫn làm được nếu chưa chốt.
- **[ĐỀ XUẤT HOÃN]**: đề xuất chuyển ra khỏi Phase 1 để vừa sức chứa (xem mục 4).

**Điểm ước lượng:** 1 / 2 / 3 / 5 (story hơn 5 điểm phải tách). **1 điểm ≈ 1,5 giờ làm việc** gồm backend, frontend và test **[GIẢ ĐỊNH, hiệu chỉnh lại sau đợt đầu tiên]**.

**Ngày giờ:** hiển thị `dd/mm/yyyy HH:mm`, múi giờ GMT+7 (theo NFR trong PRD).

**Definition of Ready (đủ điều kiện bắt đầu):**
- [ ] Đúng cú pháp, có AC kiểm chứng được, gắn một FR nguồn
- [ ] Không còn câu hỏi mở chặn việc làm (hoặc đã có giả định mặc định)
- [ ] Đã ước lượng và không quá 5 điểm

**Definition of Done (dùng chung):**
- [ ] Đáp ứng toàn bộ AC
- [ ] Có test cho logic quan trọng; story có yếu tố quyền thì có test phân quyền
- [ ] Không lộ dữ liệu nhạy cảm; lỗi trả đúng mã và thông báo rõ
- [ ] Chạy được theo hướng dẫn kiểm tra, commit theo Conventional Commits, tài liệu liên quan đã cập nhật

---

## 2. Tổng quan

| Epic | Nhóm FR | Story | Must (điểm) | Should/Could (điểm) |
|---|---|---|---|---|
| E1 Tài khoản & phân quyền | FR-AUTH | 10 | 8 story / 16 | A05 (2), A10 (3) |
| E2 Khoá học | FR-COURSE | 9 | 8 story / 21 | C09 (3) |
| E3 Đăng ký học | FR-ENROLL | 7 | 6 story / 13 | E07 (1) |
| E4 Học tập & tiến độ | FR-PROGRESS | 3 | 3 story / 8 | |
| E5 Bài tập & điểm | FR-ASSIGN | 12 | 9 story / 22 | S11 (3), S10 (3), S12 (2) |
| **Tổng** | | **41** | **34 story / 80 điểm** | 17 điểm |

### Bản đồ hành trình (story mapping)

```
Tạo TK GV ─► Dựng khoá ─────────► Đăng ký học ─► Duyệt ─► Học ──────► Nộp bài ──► Chấm điểm ──► Xuất điểm
   A07       C01 C02 C04 C05 C03     E01          E02      C07 C08      S02 S03     S06 S07       S09
             (C06 hoãn)              E04          E03      P01 P02      S05         S08
                                                  E05      P03          (S04 hoãn)
Nền chung: A01 A02 A03 A04 (đăng ký/đăng nhập/phiên) | A06 A08 A09 (quản trị) | Should: A05 C09 S11
```

---

## 3. Danh sách story

## Epic E1: Tài khoản & phân quyền

### US-A01: Đăng ký tài khoản (FR-AUTH-01, Must, 2 điểm)
Với vai trò **khách**, tôi muốn **đăng ký tài khoản** để **học các khoá học**.
- Nhập email, mật khẩu, họ tên; email chưa tồn tại
- Mật khẩu tối thiểu 8 ký tự, có chữ hoa, chữ thường và số
- Tài khoản tạo ra luôn có role STUDENT, bỏ qua mọi trường role gửi lên
- Email trùng (không phân biệt hoa/thường) thì báo lỗi rõ ràng

### US-A02: Đăng nhập (FR-AUTH-02, Must, 3 điểm)
Với vai trò **người dùng đã có tài khoản**, tôi muốn **đăng nhập** để **sử dụng hệ thống**.
- Đúng email và mật khẩu thì vào được hệ thống
- Sai email hoặc sai mật khẩu đều hiện cùng một thông báo chung
- Tài khoản bị khoá thì không đăng nhập được và có thông báo rõ
- Đăng nhập sai quá 5 lần/phút thì bị chặn tạm thời

### US-A03: Tự động gia hạn phiên (FR-AUTH-02, Must, 3 điểm)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **phiên tự gia hạn** để **không bị đăng xuất giữa chừng**.
- Phiên hết hạn ngắn thì tự gia hạn mà tôi không phải đăng nhập lại
- Mỗi mã gia hạn chỉ dùng được một lần; dùng lại mã cũ thì mọi phiên của tài khoản đó bị thu hồi
- Mã gia hạn hết hạn (7 ngày) hoặc đã thu hồi thì yêu cầu đăng nhập lại

### US-A04: Đăng xuất (FR-AUTH-02, Must, 1 điểm)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **đăng xuất** để **kết thúc phiên an toàn**.
- Đăng xuất xong, phiên hiện tại không dùng lại được
- Quay lại trang cần đăng nhập thì bị chuyển về trang đăng nhập

### US-A05: Quản lý hồ sơ và mật khẩu (FR-AUTH-05, **Should**, 2 điểm)
Với vai trò **người dùng đã đăng nhập**, tôi muốn **xem, sửa hồ sơ và đổi mật khẩu** để **giữ tài khoản chính xác và an toàn**.
- Xem được thông tin cá nhân; không bao giờ thấy mật khẩu
- Chỉ sửa được họ tên; không tự đổi email, role, trạng thái
- Đổi mật khẩu phải nhập đúng mật khẩu cũ
- Đổi mật khẩu xong, các phiên đăng nhập khác bị đăng xuất

### US-A06: Xem danh sách người dùng (FR-AUTH-03, Must, 2 điểm)
Với vai trò **ADMIN**, tôi muốn **xem danh sách user có tìm kiếm, lọc, phân trang** để **quản lý hiệu quả**.
- Lọc theo role, trạng thái; tìm theo email hoặc họ tên
- Phân trang (mặc định 20 dòng/trang, tối đa 100)
- Người không phải ADMIN nhận lỗi không đủ quyền

### US-A07: Tạo tài khoản giảng viên (FR-AUTH-03, Must, 1 điểm)
Với vai trò **ADMIN**, tôi muốn **tạo tài khoản cho giảng viên/trợ giảng** để **họ bắt đầu dựng khoá học**.
- Nhập email, họ tên, mật khẩu tạm, chọn role
- Email trùng thì báo lỗi
- Chỉ ADMIN mới thấy và dùng được chức năng này

### US-A08: Khoá/mở khoá tài khoản (FR-AUTH-03, Must, 2 điểm) *(giả định #7: chỉ khoá, không xoá thật)*
Với vai trò **ADMIN**, tôi muốn **khoá hoặc mở khoá tài khoản** để **xử lý vi phạm hoặc cấp lại quyền nhanh**.
- Khoá xong, user không đăng nhập được và phiên đang mở bị vô hiệu ngay
- ADMIN không tự khoá chính mình
- Mở khoá thì user đăng nhập lại bình thường, dữ liệu học tập vẫn còn

### US-A09: Gán role cho user (FR-AUTH-04, Must, 2 điểm) *(giả định #6: một user có nhiều role)*
Với vai trò **ADMIN**, tôi muốn **gán role cho user** để **cấp đúng quyền cho đúng người**.
- Thay toàn bộ danh sách role của user bằng danh sách mới
- Không được gỡ role ADMIN của quản trị viên cuối cùng
- Role mới có hiệu lực ở lần đăng nhập/gia hạn phiên kế tiếp

### US-A10: Chỉnh permission của role (FR-AUTH-06, **Could**, 3 điểm)
Với vai trò **ADMIN**, tôi muốn **xem và chỉnh permission của từng role** để **kiểm soát chức năng chi tiết**.
- Xem được danh sách permission và permission của mỗi role
- Thay đổi có hiệu lực sau tối đa 1 phút
- Không được gỡ quyền quản trị khỏi role ADMIN

> Lưu ý: `auth.md` đang ghi US-A10 là *Should*, PRD ghi *Could*. Cần thống nhất (đề xuất theo PRD: Could).

---

## Epic E2: Khoá học

### US-C01: Tạo khoá học (FR-COURSE-01, Must, 3 điểm) *(giả định #1: một khoá = một đợt học)*
Với vai trò **giảng viên**, tôi muốn **tạo khoá học mới** để **bắt đầu dựng nội dung cho đợt học**.
- Nhập tên (bắt buộc), mô tả, ngày bắt đầu, ngày kết thúc dự kiến, sĩ số tối đa
- Thiếu tên thì báo lỗi; ngày kết thúc không được trước ngày bắt đầu; sĩ số là số nguyên dương
- Khoá mới ở trạng thái *Nháp*, chỉ người tạo và ADMIN thấy
- Học viên không tạo được khoá

### US-C02: Xem danh sách khoá của mình và sửa thông tin (FR-COURSE-01, Must, 2 điểm)
Với vai trò **giảng viên**, tôi muốn **xem các khoá do mình phụ trách và sửa thông tin** để **quản lý nhiều khoá cùng lúc**.
- Thấy mọi khoá do mình tạo, lọc theo trạng thái, phân trang (ADMIN thấy tất cả khoá)
- Sửa được tên, mô tả, ngày, sĩ số khi khoá chưa *Đã kết thúc*
- Giảm sĩ số tối đa xuống thấp hơn số học viên đã được duyệt thì bị từ chối
- Sửa khoá của giảng viên khác bị từ chối (ADMIN được phép)

### US-C03: Công bố và kết thúc khoá (FR-COURSE-01, Must, 2 điểm)
Với vai trò **giảng viên**, tôi muốn **công bố khoá khi sẵn sàng và kết thúc khi học xong** để **học viên chỉ thấy khoá đã hoàn chỉnh**.
- Công bố được khi khoá có ít nhất 1 bài học; nếu không thì báo rõ lý do
- Khoá *Đã công bố* hiện trong danh sách của học viên
- Khoá *Đã kết thúc*: không nhận đăng ký và bài nộp mới, học viên vẫn xem được nội dung và điểm
- Chuyển khoá đã công bố về *Nháp* chỉ được khi chưa có học viên nào được duyệt

### US-C04: Quản lý chương (FR-COURSE-02, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **tạo, sửa, xoá và sắp xếp chương** để **tổ chức nội dung khoá theo trình tự học**.
- Tạo/sửa chương với tên bắt buộc; đổi thứ tự chương và thứ tự được lưu lại
- Xoá chương có bài học thì hiện cảnh báo xác nhận
- Xoá chương chứa bài học đã có tiến độ hoặc bài nộp của học viên thì bị chặn kèm thông báo
- Chỉ giảng viên của khoá (hoặc ADMIN) thao tác được

### US-C05: Quản lý bài học: nội dung và video (FR-COURSE-02, FR-COURSE-03, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **tạo bài học gồm nội dung và video** để **học viên có đủ tài liệu để học**.
- Nhập tiêu đề (bắt buộc), nội dung văn bản định dạng cơ bản (Markdown), liên kết video (tuỳ chọn)
- Liên kết video chỉ nhận YouTube/Google Drive qua https và hiển thị nhúng xem được; liên kết khác bị từ chối
- Sửa, xoá, đổi thứ tự bài học trong chương và giữa các chương
- Nội dung chứa mã script không được thực thi khi hiển thị cho học viên

### US-C06: Đính kèm tài liệu vào bài học (FR-COURSE-03, Must, 3 điểm) **[ĐỀ XUẤT HOÃN]**
Với vai trò **giảng viên**, tôi muốn **đính kèm tệp vào bài học** để **học viên tải slide, bài mẫu**.
- Nhận pdf, zip, png, jpg, md, txt; tối đa 10 MB mỗi tệp; sai loại hoặc quá lớn thì báo lỗi rõ
- Học viên đã được duyệt của khoá tải được; người ngoài khoá không tải được
- Xoá được tài liệu khỏi bài học
> Lý do đề xuất hoãn: trong Phase 1 giảng viên có thể dán liên kết Drive vào nội dung bài học (đã làm được ở US-C05); bỏ tải tệp lên sẽ tránh được phần hạ tầng lưu tệp và bảo mật tệp.

### US-C07: Xem và tìm khoá đã công bố (FR-COURSE-04, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **xem và tìm các khoá đang mở** để **chọn khoá phù hợp để đăng ký**.
- Thấy danh sách khoá *Đã công bố*, phân trang, tìm theo tên; hiển thị tên, giảng viên, thời gian, số chỗ còn lại
- Khoá *Nháp* không hiện với học viên
- Xem chi tiết khoá: mô tả và đề cương (tên chương, tên bài), chưa xem được nội dung bài khi chưa được duyệt
- Cần đăng nhập để xem

### US-C08: Xem nội dung bài học của khoá đã được duyệt (FR-COURSE-03, Must, 3 điểm)
Với vai trò **học viên**, tôi muốn **học các bài của khoá mình đã được duyệt** để **theo kịp chương trình**.
- Học viên *Đã duyệt* xem được toàn bộ chương và bài học theo đúng thứ tự, có nút bài trước/bài sau
- Trạng thái *Chờ duyệt*, *Bị từ chối*, *Đã gỡ*, hoặc chưa đăng ký thì không xem được nội dung và có thông báo rõ
- Khoá *Nháp* không xem được; khoá *Đã kết thúc* vẫn xem được
- Không đoán được nội dung bài của khoá khác bằng cách đổi đường dẫn

### US-C09: Nhân bản khoá học (FR-COURSE-05, **Should**, 3 điểm) **[CHỜ QUYẾT ĐỊNH #1]**
Với vai trò **giảng viên**, tôi muốn **nhân bản một khoá có sẵn** để **dựng khoá mới trong vài phút** (mục tiêu G3).
- Tạo bản sao tên "<tên gốc> (bản sao)" ở trạng thái *Nháp*
- Sao chép chương, bài học, bài tập (đề, thang điểm); **không** sao chép học viên, bài nộp, điểm, tiến độ
- Ngày bắt đầu/kết thúc phải nhập lại; khoá gốc không bị ảnh hưởng
- Chỉ nhân bản được khoá của mình (ADMIN nhân bản được mọi khoá)

---

## Epic E3: Đăng ký học

### US-E01: Gửi yêu cầu đăng ký khoá học (FR-ENROLL-01, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **gửi yêu cầu đăng ký một khoá đã công bố** để **được tham gia khi giảng viên chấp nhận**.
- Đã đăng nhập, bấm "Đăng ký" ở khoá đã công bố thì thấy khoá ở trạng thái *Chờ duyệt* trong "Khoá của tôi"
- Khoá chưa công bố hoặc đã kết thúc thì không đăng ký được
- Đã đăng ký khoá này rồi thì báo rõ, không tạo bản ghi thứ hai
- Chưa đăng nhập thì được yêu cầu đăng nhập trước

### US-E02: Duyệt hoặc từ chối yêu cầu đăng ký (FR-ENROLL-02, Must, 3 điểm) *(giả định #3: chỉ giảng viên tạo khoá và ADMIN)*
Với vai trò **giảng viên**, tôi muốn **duyệt hoặc từ chối yêu cầu đăng ký vào khoá của mình** để **chỉ nhận học viên đã đóng học phí**.
- Thấy danh sách yêu cầu *Chờ duyệt* của khoá mình, kèm họ tên và ngày gửi
- Duyệt thì học viên vào được nội dung khoá; từ chối thì học viên thấy trạng thái *Bị từ chối*
- Chỉ duyệt được yêu cầu của khoá do mình phụ trách (ADMIN duyệt được mọi khoá)
- Khoá đã đủ sĩ số tối đa thì không duyệt thêm và có thông báo rõ

### US-E03: Thêm học viên trực tiếp vào khoá (FR-ENROLL-02, Must, 2 điểm)
Với vai trò **giảng viên**, tôi muốn **thêm trực tiếp một học viên vào khoá** để **xử lý nhanh trường hợp đã đóng tiền mà chưa kịp đăng ký**.
- Chọn học viên có sẵn bằng email; học viên vào khoá ngay ở trạng thái *Đã duyệt*
- Học viên đã ở trong khoá thì không thêm trùng
- Khoá đủ sĩ số thì không thêm được

### US-E04: Xem các khoá của tôi và trạng thái (FR-ENROLL-03, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **xem các khoá của mình kèm trạng thái** để **biết mình đang được học khoá nào**.
- Hiển thị khoá *Chờ duyệt*, *Đã duyệt*, *Bị từ chối* (phân biệt rõ)
- Chỉ thấy khoá của chính mình
- Bấm vào khoá *Đã duyệt* thì vào học được

### US-E05: Xem danh sách học viên của khoá (FR-ENROLL-04, Must, 2 điểm)
Với vai trò **giảng viên**, tôi muốn **xem danh sách học viên trong khoá** để **nắm lớp mình có những ai**.
- Thấy học viên theo trạng thái, kèm ngày tham gia, hiển thị "số đã duyệt/sĩ số tối đa"
- Lọc theo trạng thái, tìm theo tên hoặc email
- Chỉ xem được khoá của mình (ADMIN xem mọi khoá)

### US-E06: Gỡ học viên khỏi khoá (FR-ENROLL-04, Must, 2 điểm) **[ĐỀ XUẤT HOÃN]**
Với vai trò **giảng viên**, tôi muốn **gỡ một học viên khỏi khoá** để **xử lý học viên nghỉ học hoặc nhập nhầm**.
- Gỡ xong, học viên mất quyền xem nội dung và nộp bài của khoá đó
- Bài nộp và điểm cũ được giữ nguyên, hiển thị đánh dấu *Đã gỡ*
- Duyệt lại được nếu gỡ nhầm
> Lý do đề xuất hoãn: trong Phase 1 giảng viên có thể từ chối/để nguyên; ADMIN xử lý tay khi cần. Giá trị thấp so với công sức vì phải xử lý nhiều trường hợp liên quan đến tiến độ và điểm.

### US-E07: Tự động duyệt khoá miễn phí (FR-ENROLL-01, **Could**, 1 điểm) **[CHỜ QUYẾT ĐỊNH #2]**
Với vai trò **giảng viên**, tôi muốn **đánh dấu khoá miễn phí để học viên được duyệt ngay** để **không phải duyệt tay từng người**.
- Khoá đánh dấu miễn phí thì đăng ký xong vào học luôn
- Khoá mặc định không miễn phí, vẫn qua bước duyệt
- Vẫn tuân thủ sĩ số tối đa
> Cần bổ sung trường "miễn phí" cho khoá (mở rộng FR-COURSE-01) nếu bạn chọn làm.

---

## Epic E4: Học tập & tiến độ

### US-P01: Đánh dấu hoàn thành bài học (FR-PROGRESS-01, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **đánh dấu mình đã học xong một bài** để **theo dõi mình đã đi đến đâu**.
- Đánh dấu hoặc bỏ đánh dấu hoàn thành; trạng thái được lưu và hiển thị lại khi vào lại
- Chỉ học viên *Đã duyệt* của khoá mới đánh dấu được
- Đánh dấu lặp lại không tạo dữ liệu trùng
- Khoá *Đã kết thúc* thì không thay đổi được tiến độ

### US-P02: Xem tiến độ cá nhân (FR-PROGRESS-02, Must, 3 điểm)
Với vai trò **học viên**, tôi muốn **xem phần trăm tiến độ và các bài còn thiếu** để **không bị lạc nhịp khi học** (mục tiêu G4).
- Thấy % hoàn thành = số bài đã học / tổng số bài, làm tròn đến 1%
- Thấy danh sách bài chưa hoàn thành, bấm vào là mở bài đó
- Giảng viên thêm bài mới thì % tính lại đúng
- Khoá chưa có bài học thì hiển thị "Chưa có bài học", không báo lỗi
- Chỉ xem được tiến độ của chính mình
> Bài tập chưa nộp được hiển thị ở US-S02.

### US-P03: Xem tiến độ cả lớp (FR-PROGRESS-03, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **xem tiến độ từng học viên trong lớp** để **phát hiện sớm học viên bỏ dở**.
- Bảng học viên *Đã duyệt* gồm % hoàn thành và lần học gần nhất
- Sắp xếp theo % tăng/giảm; tìm theo tên
- Số liệu mỗi học viên khớp với trang tiến độ cá nhân của học viên đó
- Chỉ xem được khoá của mình (ADMIN xem mọi khoá); học viên không truy cập được

---

## Epic E5: Bài tập & điểm

### US-S01: Tạo và sửa bài tập (FR-ASSIGN-01, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **giao bài tập cho khoá** để **học viên biết phải làm gì và nộp khi nào**.
- Nhập tiêu đề, đề bài (Markdown), hạn nộp (`dd/mm/yyyy HH:mm`, GMT+7), điểm tối đa (mặc định 10), tuỳ chọn gắn với một bài học
- Hạn nộp phải ở tương lai khi tạo; thiếu tiêu đề hoặc hạn nộp thì báo lỗi
- Sửa được bài tập; đổi hạn nộp khi đã có bài nộp thì hiện cảnh báo
- Xoá bài tập đã có bài nộp thì bị chặn
- Bài tập mới tạo hiển thị ngay với học viên *Đã duyệt* của khoá

### US-S02: Xem bài tập và hạn nộp (FR-ASSIGN-02, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **xem danh sách bài tập của khoá kèm trạng thái** để **biết còn bài nào chưa nộp**.
- Mỗi bài tập có trạng thái: *Chưa nộp*, *Đã nộp*, *Đã chấm*, *Quá hạn* (chưa nộp mà đã hết hạn)
- Sắp xếp theo hạn nộp; hiển thị thời gian còn lại hoặc đã quá hạn
- Chỉ học viên *Đã duyệt* của khoá xem được

### US-S03: Nộp bài bằng liên kết hoặc ghi chú (FR-ASSIGN-02, Must, 2 điểm) *(giả định #5: khoá cứng sau hạn)*
Với vai trò **học viên**, tôi muốn **nộp bài bằng liên kết repo và ghi chú** để **giảng viên có bài của tôi để chấm**.
- Nhập liên kết (https) và ghi chú tuỳ chọn; liên kết sai định dạng thì báo lỗi
- Nộp trước hạn thành công thì trạng thái thành *Đã nộp* và ghi nhận thời điểm nộp
- Sau hạn nộp thì không nộp được và có thông báo rõ
- Chưa được duyệt hoặc đã bị gỡ khỏi khoá thì không nộp được

### US-S04: Nộp bài bằng tệp (FR-ASSIGN-02, Must, 3 điểm) **[ĐỀ XUẤT HOÃN]**
Với vai trò **học viên**, tôi muốn **nộp bài bằng tệp** để **nộp được cả bài không có repo**.
- Nhận pdf, zip, png, jpg, md, txt; tối đa 10 MB mỗi tệp, tối đa 5 tệp mỗi lần nộp
- Sai loại hoặc quá lớn thì báo lỗi rõ
- Chỉ giảng viên của khoá và chính học viên đó tải được tệp
> Lý do đề xuất hoãn: với khoá lập trình, liên kết repo (GitHub) là hình thức nộp chủ yếu; hoãn cùng US-C06 giúp bỏ hẳn phần hạ tầng lưu tệp trong Phase 1.

### US-S05: Nộp lại trước hạn (FR-ASSIGN-02, Must, 1 điểm)
Với vai trò **học viên**, tôi muốn **nộp lại bài trước hạn** để **sửa lỗi mà không phải nhờ giảng viên**.
- Nộp lại thì bài mới thay thế bài cũ và thời điểm nộp được cập nhật
- Bài đã được chấm thì không nộp lại được
- Hết hạn thì không nộp lại được

### US-S06: Xem bài nộp theo bài tập (FR-ASSIGN-03, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **xem toàn bộ bài nộp của một bài tập** để **tìm và chấm bài nhanh** (mục tiêu G1).
- Thấy mọi học viên *Đã duyệt* kèm trạng thái *Chưa nộp / Đã nộp / Đã chấm*, thống kê "đã nộp x/y"
- Lọc theo trạng thái; tìm bài của một học viên theo tên và mở ra dưới 1 phút
- Mở chi tiết thấy liên kết, ghi chú, thời điểm nộp
- Chỉ xem được khoá của mình (ADMIN xem mọi khoá); học viên không truy cập được

### US-S07: Chấm điểm và nhận xét (FR-ASSIGN-04, Must, 3 điểm)
Với vai trò **giảng viên**, tôi muốn **chấm điểm và nhận xét bài nộp** để **học viên biết kết quả và cách cải thiện**.
- Nhập điểm từ 0 đến điểm tối đa của bài (tối đa 1 chữ số thập phân) và nhận xét tuỳ chọn
- Điểm ngoài thang bị từ chối kèm thông báo rõ
- Không chấm được học viên chưa nộp bài
- Sửa lại điểm sau khi chấm được; hệ thống ghi nhận thời điểm chấm gần nhất
- Học viên không chấm được

### US-S08: Xem điểm và nhận xét của mình (FR-ASSIGN-04, Must, 2 điểm)
Với vai trò **học viên**, tôi muốn **xem điểm và nhận xét của từng bài** để **biết mình làm tốt hay cần sửa**.
- Bài đã chấm hiển thị điểm và nhận xét; bài chưa chấm hiển thị "Chưa chấm"
- Xem được bảng điểm các bài của mình trong khoá
- Không xem được điểm hay bài nộp của học viên khác (phải được kiểm tra bằng test)

### US-S09: Xuất bảng điểm khoá (FR-ASSIGN-05, Must, 3 điểm) **[CHỜ QUYẾT ĐỊNH #4]** *(mặc định: điểm tổng = trung bình cộng)*
Với vai trò **giảng viên**, tôi muốn **xuất bảng điểm của khoá** để **có điểm cuối khoá trong vài phút thay vì tự tổng hợp** (mục tiêu G2).
- Xuất tệp CSV (UTF-8 có BOM) mở đúng tiếng Việt trong Excel: mỗi học viên một dòng, mỗi bài tập một cột, thêm cột điểm tổng
- Điểm tổng mặc định = trung bình cộng điểm các bài đã giao (quy về thang 10); bài chưa nộp tính 0
- Chỉ gồm học viên *Đã duyệt*; ngày giờ theo `dd/mm/yyyy`
- Chỉ xuất được khoá của mình (ADMIN xuất mọi khoá)
> Xuất Excel (.xlsx) để sau nếu cần; CSV đủ để mở trong Excel.

### US-S10: Cấu hình trọng số điểm cuối khoá (FR-ASSIGN-05, **Could**, 3 điểm) **[CHỜ QUYẾT ĐỊNH #4]**
Với vai trò **giảng viên**, tôi muốn **đặt trọng số cho từng bài tập** để **điểm cuối khoá phản ánh đúng tầm quan trọng** (ví dụ bài tập 40%, đồ án 60%).
- Đặt trọng số theo bài hoặc theo nhóm bài; tổng trọng số phải bằng 100%
- Điểm tổng trong bảng điểm và khi xuất tính theo trọng số đã đặt
- Chưa đặt trọng số thì dùng trung bình cộng như US-S09

### US-S11: Thông báo trong hệ thống (FR-ASSIGN-06, **Should**, 3 điểm)
Với vai trò **học viên hoặc giảng viên**, tôi muốn **nhận thông báo khi có sự kiện liên quan** để **không phải liên tục mở từng khoá để kiểm tra**.
- Học viên nhận thông báo khi có bài tập mới và khi bài được chấm; giảng viên nhận khi có bài nộp mới
- Có biểu tượng chuông với số thông báo chưa đọc; đánh dấu đã đọc được
- Chỉ thấy thông báo của chính mình

### US-S12: Cho phép nộp trễ (FR-ASSIGN-02, **Could**, 2 điểm) **[CHỜ QUYẾT ĐỊNH #5]**
Với vai trò **giảng viên**, tôi muốn **cho phép nộp trễ có trừ điểm** để **linh hoạt với học viên gặp sự cố**.
- Bật/tắt nộp trễ theo từng bài tập, đặt tỷ lệ trừ điểm
- Bài nộp trễ được đánh dấu rõ cho giảng viên khi chấm
- Mặc định tắt (khoá cứng sau hạn như US-S03)

---

## 4. Kiểm tra sức chứa

PRD: khoảng 180 giờ (15 giờ/tuần × 12 tuần), chừa 30% cho lỗi phát sinh, còn **~126 giờ** dùng cho cả **khởi tạo dự án (~22 giờ)** và **tính năng**.

| Phương án | Điểm Must | Giờ tính năng (×1,5) | + Setup | So với 126 giờ |
|---|---|---|---|---|
| Giữ nguyên toàn bộ Must trong PRD | 80 | 120 | ~142 | **Vượt ~16 giờ** |
| Hoãn 3 story đề xuất (C06, S04, E06) | 72 | 108 | ~130 | Vượt ~4 giờ (sát giới hạn, chấp nhận được) |

**Đề xuất:** chọn phương án hoãn 3 story (tổng 8 điểm). Nếu đồng ý, cần cập nhật PRD lên v0.2:
- FR-COURSE-03: bỏ "tài liệu đính kèm" khỏi Phase 1 (dùng liên kết trong nội dung)
- FR-ASSIGN-02: Phase 1 chỉ nộp bằng liên kết repo/ghi chú, nộp tệp sang Phase 2
- FR-ENROLL-04: Phase 1 chỉ xem danh sách học viên, gỡ học viên sang Phase 2

Sau đợt đầu tiên, đo lại tốc độ thực tế (điểm làm được mỗi đợt) rồi điều chỉnh hệ số 1,5 giờ/điểm.

---

## 5. Thứ tự thực hiện (giả sử đã chọn phương án hoãn 3 story)

Mỗi đợt 2 tuần ≈ 30 giờ, trừ 30% dự phòng ≈ 21 giờ ≈ 14 điểm. Đợt 1 dành cho giai đoạn **Setup** (~22 giờ).

| Đợt | Mục tiêu | Story | Điểm |
|---|---|---|---|
| 1 | Setup và nền móng chạy được | (Giai đoạn 4: repo, NestJS, React, DB, CI cơ bản) | |
| 2 | **Walking skeleton**: đăng nhập, tạo khoá, thấy khoá trên giao diện | A01, A02, A03, A04, C01, C07 | 14 |
| 3 | Quản trị người dùng, dựng cấu trúc khoá | A06, A07, A08, A09, C02, C03, C04 | 14 |
| 4 | Bài học và đăng ký học | C05, C08, E01, E02, E03, E04 | 15 |
| 5 | Tiến độ, danh sách lớp, bắt đầu bài tập | E05, P01, P02, P03, S01, S02 | 15 |
| 6 | Nộp bài, chấm điểm, xuất điểm | S03, S05, S06, S07, S08, S09 | 14 |
| Dự phòng | Làm nếu còn thời gian, theo thứ tự | A05, S11, C09, rồi các story Could | |

**Lý do sắp xếp:**
- Theo phụ thuộc: AUTH → COURSE → ENROLL → PROGRESS → ASSIGN
- Đợt 2 là walking skeleton: đi xuyên mọi tầng (React → NestJS → SQL Server) với ít chức năng nhất để lộ sớm lỗi kiến trúc và phân quyền
- Phân quyền theo chủ sở hữu (C08, E02, S06, S08) nằm ở đợt 4-6 là phần rủi ro cao nên cần test kỹ; phần nền của nó đã có từ đợt 2-3

---

## 6. Quyết định đang chờ và giả định mặc định

| # | Câu hỏi (từ PRD) | Story bị ảnh hưởng | Giả định mặc định đang dùng |
|---|---|---|---|
| 1 | Tách "khoá mẫu" và "lớp"? | C01, C09 | Một khoá = một đợt; nhân bản để mở đợt mới |
| 2 | Khoá miễn phí tự duyệt? | E07 | Mọi đăng ký đều qua bước duyệt |
| 3 | Quyền của trợ giảng | E02, E05, P03, S06, S07 | Chỉ giảng viên tạo khoá và ADMIN thao tác trên khoá đó |
| 4 | Cách tính điểm cuối khoá | S09, S10 | Trung bình cộng, bài chưa nộp tính 0 |
| 5 | Nộp trễ | S03, S12 | Khoá cứng sau hạn |
| 6 | Một user nhiều role? | A02, A09 | Có, dùng bảng nối user-role |
| 7 | Xoá user thật hay chỉ khoá? | A08 | Chỉ khoá |

**Ghi chú về #3:** nếu muốn trợ giảng chấm bài ở khoá do giảng viên khác tạo, cần thêm cơ chế "gán người phụ trách khoá". PRD hiện chưa có FR cho việc này, nên chưa có story; nếu bạn chọn làm, bổ sung FR vào PRD trước.

---

## 7. Bảng truy vết FR → Story

| FR | Ưu tiên | Story |
|---|---|---|
| FR-AUTH-01 | Must | A01 |
| FR-AUTH-02 | Must | A02, A03, A04 |
| FR-AUTH-03 | Must | A06, A07, A08 |
| FR-AUTH-04 | Must | A09 |
| FR-AUTH-05 | Should | A05 |
| FR-AUTH-06 | Could | A10 |
| FR-COURSE-01 | Must | C01, C02, C03 |
| FR-COURSE-02 | Must | C04, C05 |
| FR-COURSE-03 | Must | C05, C06, C08 |
| FR-COURSE-04 | Must | C07 |
| FR-COURSE-05 | Should | C09 |
| FR-ENROLL-01 | Must | E01, E07 |
| FR-ENROLL-02 | Must | E02, E03 |
| FR-ENROLL-03 | Must | E04 |
| FR-ENROLL-04 | Must | E05, E06 |
| FR-PROGRESS-01 | Must | P01 |
| FR-PROGRESS-02 | Must | P02 |
| FR-PROGRESS-03 | Must | P03 |
| FR-ASSIGN-01 | Must | S01 |
| FR-ASSIGN-02 | Must | S02, S03, S04, S05, S12 |
| FR-ASSIGN-03 | Must | S06 |
| FR-ASSIGN-04 | Must | S07, S08 |
| FR-ASSIGN-05 | Must | S09, S10 |
| FR-ASSIGN-06 | Should | S11 |

**Kiểm tra hai chiều:** mọi FR Must đều có ít nhất một story; mọi story đều trỏ về một FR.

---

## 8. Cần đồng bộ với tài liệu khác

- `01-prd.md`: nếu chấp nhận hoãn C06, S04, E06 thì nâng lên v0.2 và ghi vào change log
- `auth.md`, `auth-user-stories.md`, `auth-spec.md`, `auth-prd.md`: đổi mã yêu cầu `FR-01...FR-10` thành `FR-AUTH-01...FR-AUTH-06` theo bảng truy vết ở trên, và thống nhất ưu tiên của US-A10
- Bản ví dụ epic Đăng ký học đã đưa ra trong phần hướng dẫn trước: US-E04 chỉ còn gắn FR-ENROLL-03; phần FR-ENROLL-04 (xem danh sách học viên) tách thành US-E05, US-E06
- Mã story trong sơ đồ minh hoạ ở phần hướng dẫn trước chỉ mang tính minh hoạ; mã chính thức là mã trong tài liệu này

## 9. Change log

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 0.1 | 09/10/2026 | Bản nháp đầu tiên: 41 story cho 5 epic, kiểm tra sức chứa, thứ tự thực hiện |
