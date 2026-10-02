# AI Context — Kế hoạch chuyển đồ án quản lý vật liệu xây dựng sang PHP

> **Trạng thái:** Tài liệu kế hoạch được đưa lên repository PHP riêng theo yêu cầu; mã nguồn PHP chưa được chuyển.
> **Ngày khảo sát:** 02/10/2026 (Asia/Bangkok).
> **Nguồn khảo sát:** Git root `Construction_materials_management_system/`, ứng dụng ở `ht_vlxd/`, nhánh `main`, commit `09a1413`.
> **Repository đích:** `Construction_materials_management_system_php`.

## 1. Mục tiêu và nguyên tắc

Chuyển toàn bộ nghiệp vụ có thể sử dụng của ứng dụng Java Spring Boot sang PHP, dùng **Laravel 13 + PHP 8.3 + MySQL + Blade + Bootstrap 5.3** làm thiết kế mặc định. PHP 8.3 đang có trên máy; Composer chưa có trong PATH, vì vậy Phase 1 phải kiểm tra/cài môi trường trước khi tạo ứng dụng. Laravel 13 yêu cầu PHP 8.3 trở lên theo [release notes chính thức](https://laravel.com/framework/docs/releases); Bootstrap 5.3 có [hướng dẫn chính thức](https://getbootstrap.com/docs/5.3/getting-started/introduction/). Khóa phiên bản chính xác trong `composer.lock` và lockfile frontend khi bắt đầu triển khai.

Giữ tên bảng/cột và dữ liệu MySQL hiện có nếu đã triển khai thực tế; thay đổi schema bằng migration có kiểm soát. Thiết kế lại giao diện bằng Bootstrap 5, giữ đầy đủ luồng **đã có trong code** và thông điệp tiếng Việt. Các tính năng chỉ được mô tả trong README nhưng chưa thấy triển khai phải được quyết định riêng trong Phase 0, không mặc nhiên coi là phạm vi chuyển tương đương. Mỗi phase tạo được một phần chạy và kiểm thử được. Không chuyển cơ chế “demo role” ở `localStorage` thành cơ chế phân quyền thật.

## 2. Ảnh chụp hiện trạng đã xác minh

| Thành phần | Hiện trạng từ mã nguồn | Điểm cần chuyển |
|---|---|---|
| Nền tảng | Spring Boot `4.0.6`, Java `26`, Spring Web MVC, Data JPA, Security, Thymeleaf; MySQL Connector/J (`pom.xml`) | Laravel controllers, Eloquent/repositories theo nhu cầu, Blade, MySQL |
| Cơ sở dữ liệu | Git root có `Database.sql` (18 `CREATE TABLE` + seed + index); ứng dụng có `Migration_Material_Estimation.sql` cho bảng định mức thứ 19 và thay đổi đơn hàng; `spring.jpa.hibernate.ddl-auto=update` | Đối chiếu **cả hai SQL**, entity và schema DB thực tế trước khi tạo baseline migration |
| Quy mô | 19 `@Entity`, 19 repository, 17 service, 9 controller, 70 REST endpoint, 24 mapping ở view controller, 22 template HTML (19 trang và 3 fragment), 3 file test Java | Lập ma trận chuyển đổi theo module và endpoint |
| Giao diện | Thymeleaf + CSS tự viết + JavaScript inline/fetch; không thấy import Bootstrap trong template | Blade layouts/components + Bootstrap 5.3 + JS module; vẫn dùng CSS riêng cho nhận diện |
| Tích hợp | `/api/ai/chat` gọi Gemini; chat UI trong `static/js/custom-alert.js` | HTTP client phía server, key trong `.env`, giới hạn truy cập/tần suất, xử lý timeout |
| Dữ liệu mẫu | `Database.sql` có seed ban đầu; `DatabaseSeeder` tạo 8 vai trò, 6 tài khoản mẫu, danh mục, hàng hóa, kho, tồn kho, định mức và bổ sung cát/đá | Seeders chỉ dành cho môi trường dev/test, chạy tường minh, không đặt lại mật khẩu khi khởi động; hợp nhất nguồn seed |
| Kiểm thử | 1 context test, 1 customer service test, 1 estimation service test | Feature/unit tests Laravel và bộ kiểm thử đối chiếu hành vi |

**Giới hạn khảo sát:** Đã kiểm kê toàn bộ **113 file được Git theo dõi** ở root, gồm `Database.sql`, `README.md`, `README_EN.md` mà lượt khảo sát đầu chưa đọc; file kế hoạch này chưa được Git theo dõi **trong repository nguồn**. Có schema **dự kiến** trong SQL, nhưng chưa có bản dump/schema **MySQL đang chạy**, dữ liệu thực tế, tài khoản triển khai hay môi trường web mục tiêu. Chưa thể xác nhận drift giữa SQL/JPA và DB hoặc chạy kiểm thử tích hợp với MySQL. `mvnw.cmd test -q` đã thử bằng PowerShell và `cmd.exe`, nhưng wrapper dừng trước khi khởi chạy Maven (`Cannot index into a null array`). `mvnw.cmd` truy cập `.Target[0]` của thư mục `.m2` thường; `.Target` ở máy này là null, phù hợp với lỗi quan sát được. Đây là giới hạn của kiểm thử hiện tại, không phải kết luận test pass/fail. `application.properties` đang được Git theo dõi; trường mật khẩu DB có giá trị không rỗng và không giống placeholder (không hiển thị tại đây), còn trường Gemini trông giống placeholder. Phải xác minh và thay mật khẩu DB nếu còn dùng trước khi công khai repo PHP; không sao chép file này.

### 2.1 Thứ tự ưu tiên bằng chứng

Khi các nguồn mâu thuẫn: **DB đang chạy (chưa có quyền kiểm tra) → controller/service/entity hiện tại → SQL trong repo → README**. `README.md`/`README_EN.md` mô tả hợp đồng có giá cố định theo từng vật tư, kiểm tra hạn mức nợ trước xuất kho và RBAC/CSRF. Mã hiện tại chưa có bảng chi tiết giá hợp đồng, không thấy phép kiểm `hanMucNo` ở luồng duyệt/xuất, còn `SecurityConfig` mở mọi request và tắt CSRF. Đây là **chênh lệch giữa tài liệu và triển khai**, cần người dùng chọn “bảo toàn code hiện tại” hay “bổ sung theo đặc tả README” ở Phase 0. README ghi 20 entity, nhưng đếm `@Entity` cho ra 19; header `Database.sql` ghi 16 bảng, nhưng file có 18 lệnh `CREATE TABLE`.

### 2.2 Phạm vi file đã kiểm kê

| Vị trí | Số file Git theo dõi | Nội dung |
|---|---:|---|
| Git root | 3 | `Database.sql`, README tiếng Việt và tiếng Anh |
| `ht_vlxd/` hạ tầng | 6 | Maven, wrapper, Git config |
| `ht_vlxd/src/main/java/` | 72 | 2 config, 9 controller, 4 DTO, 1 entrypoint, 20 model file (19 entity + 1 enum), 19 repository, 17 service |
| `ht_vlxd/src/main/resources/` | 29 | 22 template, 2 SQL/config, CSS, JS chat/alert, 2 script Python hỗ trợ, 1 ảnh upload mẫu |
| `ht_vlxd/src/test/` | 3 | 3 lớp test Java |
| **Tổng** | **113** | Toàn bộ danh sách từ `git ls-files`; chưa có database sống hoặc file nằm ngoài repo |

`Database.sql` có seed mật khẩu dạng chuỗi ví dụ `examplehash`, không phải hash BCrypt dùng được; seeder Java mới tạo hash thực khi chạy. Vì vậy không nhập nguyên seed SQL đó vào môi trường PHP rồi coi đăng nhập mẫu là đã hoạt động. Hai script Python trong `static/js/` là công cụ trích nội dung từ một file đồ án ở đường dẫn máy khác, không phải JavaScript chạy trên web; file đầu vào đó không có trong repository. Ảnh trong `static/images/uploads/` là ảnh sản phẩm đã được Git theo dõi. Khi chuyển PHP, quyết định giữ script như công cụ phát triển hay loại khỏi gói deploy, và chuyển file upload sang storage/public phù hợp.

## 3. Bản đồ nghiệp vụ và dữ liệu

### 3.1 Nhóm bảng và quan hệ chính

| Module | Bảng hiện có | Quan hệ/chức năng cần giữ |
|---|---|---|
| Tài khoản | `role`, `nguoi_dung`, `khach_hang`, `nha_cung_cap` | user→role; hồ sơ khách/NCC→user; 8 role được seed |
| Sản phẩm | `danh_muc`, `hang_hoa`, `dinh_muc_vat_lieu` | danh mục cha/con; giá bán lẻ/sỉ; định mức theo loại công trình/vật liệu |
| Bán hàng | `don_hang`, `don_hang_chi_tiet`, `hop_dong`, `doi_tra_hang`, `giao_nhan` | đơn→chi tiết; khách vãng lai có thể không có `khach_hang_id`; hợp đồng/đổi trả/giao nhận gắn đơn |
| Kho | `kho`, `ton_kho`, `phieu_kho`, `phieu_kho_chi_tiet` | duy nhất `(hang_hoa_id, kho_id)`; phiếu nhập/xuất, duyệt phiếu, điều chỉnh, bàn giao |
| Tài chính | `cong_no`, `thanh_toan`, `bao_cao` | nợ gắn khách/đơn/hợp đồng; thanh toán gắn nợ; báo cáo có người lập/duyệt |

Các cột tiền dùng `DECIMAL`/`BigDecimal`, số lượng có độ chính xác 3 chữ số thập phân tại chi tiết đơn/phiếu kho. Khi chuyển sang PHP dùng decimal/string hoặc thư viện số thập phân phù hợp; không dùng float để tính tiền. Ngày giờ, timezone, collation `utf8mb4`, khóa ngoại, chỉ mục duy nhất, giá trị null/default và enum/status phải đối chiếu schema thực trước khi viết migration. `Database.sql` có `cong_no.so_tien_con_lai` là cột tính toán, `hang_hoa.ngay_cap_nhat` và `hop_dong.ngay_cap_nhat` không nằm trong entity tương ứng; cần quyết định giữ/cấp quyền ghi theo DB thực.

### 3.2 Vai trò và vùng chức năng

`QUAN_TRI_VIEN` quản lý tài khoản; `BAN_QUAN_LY` xem dashboard/báo cáo và duyệt; `NV_KINH_DOANH` quản lý đơn, hợp đồng, hàng hóa và đổi trả; `NV_KHO` quản lý kho, phiếu, giao nhận; `NV_KE_TOAN` công nợ, thanh toán, NCC, báo cáo; `KHACH_HANG` xem hàng, giỏ, hồ sơ, đơn, đổi trả. Hai role đã seed là `NV_GIAO_NHAN` và `NHA_CUNG_CAP` chưa có trang điều khiển riêng trong template; xác nhận yêu cầu trước khi mở rộng chức năng.

### 3.3 Ma trận route hiện có

| Controller | Số endpoint | Các nhóm hành vi cần đối chiếu | Phase chịu trách nhiệm |
|---|---:|---|---|
| `CustomerRestController` | 12 | hàng hóa, hồ sơ, đăng ký/đăng nhập, đổi mật khẩu, đơn, hủy, đổi trả, checkout giỏ | 3, 4, 5, 7 |
| `SalesRestController` | 14 | duyệt/hủy/hoàn tất đơn, tạo đơn, đổi trả, hợp đồng | 5, 7 |
| `WarehouseRestController` | 11 | tổng quan/tồn, phiếu, điều chỉnh, xuất đơn, giao nhận/bàn giao/sự cố | 6 |
| `AccountingRestController` | 11 | công nợ, thu tiền, thanh toán, NCC, doanh thu, dòng tiền, báo cáo | 7, 8 |
| `ManagementRestController` | 9 | dashboard, đơn, hàng chờ duyệt, tồn, duyệt hợp đồng/đổi trả/báo cáo/phiếu | 5, 6, 7, 8 |
| `AdminRestController` | 9 | danh sách, vai trò, tạo/sửa/xóa, khóa, đổi role, reset mật khẩu | 3 |
| `MaterialEstimationRestController` | 3 | tính vật liệu, chọn sản phẩm, endpoint tên “đặt hàng nháp” nhưng lưu đơn ở trạng thái `CHO_XAC_NHAN` | 9 |
| `AiChatRestController` | 1 | tư vấn AI | 9 |
| `TemplateViewController` | 24 mapping | Trang công khai, trang từng role và 4 thao tác POST quản lý sản phẩm/danh mục | 3–10 |

Giữ ma trận endpoint chi tiết bằng `php artisan route:list` khi triển khai. Có thể đổi URI trong PHP, nhưng phải cập nhật tất cả `fetch()` trong các template hoặc cung cấp route tương thích; ghi lại đối chiếu cũ→mới và contract request/response.

### 3.4 Luồng nghiệp vụ có ràng buộc

1. **Đơn hàng:** khách/giỏ hàng hoặc nhân viên tạo đơn và chi tiết; giá và tổng tiền phải tính lại trên server từ sản phẩm hợp lệ. Code dùng `CHO_XAC_NHAN`, `DA_XAC_NHAN`, `DANG_GIAO`, `HOAN_THANH`, `DA_HUY`; `Database.sql` lại khai báo `HUY` thay `DA_HUY` và có các trạng thái khác. Duyệt/hủy/hoàn tất phải có điều kiện chuyển trạng thái và kiểm tra chủ sở hữu.
2. **Hợp đồng:** tạo từ đơn được duyệt, code tính giá trị từ tổng đơn và phần trăm chiết khấu, dùng `NHAP`→`HIEU_LUC` hoặc `DA_HUY`, nhưng SQL enum dùng `HUY`. Đường duyệt ở quản lý cũng có thể kích hoạt. Xác định một luồng duyệt chuẩn và trách nhiệm từng role, kiểm tra biên chiết khấu/đặt cọc. Chưa có cấu trúc giá cố định từng vật tư như README mô tả.
3. **Kho/giao nhận:** phiếu nháp chờ quản lý duyệt, tồn kho tăng/giảm khi duyệt; giao nhận chuyển `DANG_GIAO`→`DA_GIAO`/`THAT_BAI`. Code hiện trừ tồn khi duyệt phiếu xuất **và** khi bàn giao thành công, nên cùng đơn có nguy cơ bị trừ hai lần. Cần chọn duy nhất một thời điểm ghi biến động và chống ghi lặp.
4. **Đổi trả:** yêu cầu `CHO_DUYET`→`DA_DUYET`/`TU_CHOI`; duyệt tác động cả công nợ và tồn. Hai đường duyệt ở sales và management đều lấy `hang_hoa.gia_ban_le` **hiện tại** để giảm nợ, có thể khác giá đã bán trong `don_hang_chi_tiet`. Cần chuẩn hóa một service có transaction/idempotency và quy tắc giá hoàn trả.
5. **Công nợ/thanh toán:** hoàn tất đơn có thể tạo nợ; thu tiền tạo bản ghi thanh toán rồi cập nhật số đã trả/trạng thái. Code dùng `CHUA_THANH_TOAN` nhưng SQL enum nợ chỉ có `CON_NO`, `DA_THANH_TOAN`, `QUA_HAN`. Khóa bản ghi nợ khi thu đồng thời, chặn thu vượt và không tạo nợ trùng.
6. **Ước tính vật liệu:** 3 loại công trình × 5 loại vật liệu do seeder tạo; hệ số `he_so_m2`, `THEO_TANG`/`MOT_LAN`, làm tròn lên số nguyên hoặc 0,1 đơn vị; chọn sản phẩm đúng nhóm; giá sỉ cho doanh nghiệp trong estimator/draft; khách chưa đăng nhập không nhận giá. Tuy nhiên các endpoint đặt hàng thông thường của khách và sales đang dùng `gia_ban_le` kể cả khi khách là doanh nghiệp; phải chốt chính sách giá thống nhất trước khi chuyển.

## 4. Phát hiện cần xử lý trước hoặc trong lúc chuyển

| Ưu tiên | Phát hiện có chứng cứ | Xử lý trong PHP |
|---|---|---|
| P0 | `SecurityConfig` dùng `anyRequest().permitAll()` và tắt CSRF; header có bộ chọn role demo, tự gửi thông tin đăng nhập mẫu từ JavaScript; frontend giữ `demo_role`/username trong `localStorage` | Bỏ bộ chọn role/mật khẩu demo khỏi bản PHP; session auth thật, middleware role/policy trên **mọi** route và API, CSRF, cookie an toàn; UI chỉ đọc danh tính từ server |
| P0 | Các endpoint hồ sơ/đơn dùng `username` từ query/body; đổi hồ sơ/mật khẩu cũng tìm theo username gửi từ client (`CustomerRestController`) | Lấy người dùng từ session, kiểm tra ownership; không nhận user ID/role làm nguồn quyền |
| P0 | `DatabaseSeeder.resetPasswordsToBCrypt()` ghi lại mật khẩu của tài khoản mẫu mỗi lần khởi động | Bỏ cơ chế reset tự động; tách seed dev, buộc đổi password nếu nhập tài khoản mẫu |
| P0 | Admin tạo tài khoản và reset mật khẩu về cùng một mật khẩu mặc định, còn trả mật khẩu đó qua API; trang đăng nhập cũng công khai các tài khoản mẫu | Tạo mật khẩu tạm ngẫu nhiên hoặc luồng đặt mật khẩu một lần, buộc đổi khi đăng nhập; chỉ seed tài khoản demo ở môi trường dev |
| P0 | `application.properties` được theo dõi; mật khẩu DB không rỗng/không giống placeholder; Gemini key trông giống placeholder | Kiểm tra lịch sử Git và thay credential DB nếu còn dùng; xác minh mọi key khác; `.env` không commit; `.env.example` chỉ placeholder |
| P0 | `Database.sql` có 18 bảng, migration định mức thêm bảng 19; enum trạng thái đơn/hợp đồng/nợ trong SQL không khớp chuỗi code; SQL ban đầu thiếu cột khách vãng lai và FK `phieu_kho.don_hang_id` | Lập bảng sai khác SQL↔entity↔DB; xác định schema đích, migration chuyển dữ liệu và test cho từng trạng thái |
| P0 | Duyệt phiếu xuất (`ManagementRestController`) và xác nhận bàn giao (`WarehouseRestController`) đều trừ `ton_kho` | Chọn điểm trừ kho duy nhất; test duyệt→bàn giao và gọi lặp để tồn chỉ giảm đúng một lần |
| P0 | Thu tiền lấy `username` từ request và nếu không tìm thấy thì gán người thu là user đầu tiên trong DB | Lấy người thu từ session đã xác thực, yêu cầu role kế toán, bỏ fallback danh tính |
| P0 | Các nút đăng xuất hiện chỉ xóa `localStorage` rồi chuyển trang, không thấy endpoint hủy session/security context | Tạo POST logout thật, invalidate session và CSRF token, kiểm tra request sau logout bị từ chối |
| P1 | Nhiều thao tác tiền/kho/đổi trả ghi qua nhiều repository nhưng controller không đánh dấu `@Transactional` | Service transaction, row lock hoặc optimistic lock, idempotency key, constraint và rollback test |
| P1 | Cả sales và management duyệt trả hàng theo giá bán lẻ hiện tại; `AccountingRestController.collectPayment` có thể nhận số âm/ghi phiếu 0 và tự cắt số thu vượt | Đối chiếu giá gốc đơn, kiểm tra số lượng/tiền dương, trả lỗi rõ khi vượt nợ, không ghi phiếu 0/âm |
| P1 | Upload ảnh hàng hóa ghi vào `src/main/resources/static/images/uploads` và `target/classes/...` trong `TemplateViewController` | Kiểm kê cả file upload ngoài Git trước khi chuyển, validate loại/kích cỡ file, lưu vào Laravel storage và bảo đảm URL cũ được ánh xạ |
| P1 | `TemplateViewController` trỏ tới `dung_chung/calculator` nhưng thiếu `calculator.html` | Làm rõ thiết kế màn tính vật liệu và bổ sung Blade view/API test |
| P1 | `spring.jpa.hibernate.ddl-auto=update` cùng SQL thủ công và ALTER trong seeder | Chụp DB thực, chuẩn hóa migration có thứ tự; không chạy migration mù trên DB thật |
| P1 | README nêu giá cố định theo hợp đồng và chặn vượt hạn mức nợ, nhưng code/schema chưa thể hiện đầy đủ hai luồng này | Quyết định phạm vi với người dùng: chuyển đúng chức năng hiện hành hoặc bổ sung nghiệp vụ mới, rồi thiết kế bảng/rule/test riêng |
| P1 | Estimator/draft dùng giá sỉ cho khách doanh nghiệp, nhưng đặt hàng trực tiếp/giỏ/sales dùng giá lẻ | Chốt ma trận giá theo loại khách và kênh tạo đơn; dùng một pricing service và golden tests |
| P1 | Kế toán lưu báo cáo trực tiếp `DA_DUYET`, nhưng danh sách chờ duyệt của quản lý chỉ lấy `CHO_DUYET`; sales cũng có thể kích hoạt hợp đồng trực tiếp ngoài đường duyệt quản lý | Chốt quy trình phê duyệt và ma trận quyền cho báo cáo/hợp đồng; kiểm thử luồng từ tạo tới duyệt/hủy |
| P1 | Nhiều mã khách/đơn/hợp đồng/phiếu thu/đổi trả được tạo từ `System.currentTimeMillis()` hoặc `count+1` | Dùng sequence/unique constraint/transaction và xử lý va chạm; kiểm thử tạo đồng thời |
| P1 | Nghiệp vụ chủ yếu nằm trong controller; đa số service là CRUD wrapper, ngoại lệ nổi bật là `MaterialEstimationService` | Chuyển quy tắc sang service PHP có transaction, không sao chép controller nguyên khối |
| P1 | `/api/vat-lieu/dat-hang-nhap` lưu header đơn trước khi kiểm tra hết chi tiết, không có transaction và dùng trạng thái `CHO_XAC_NHAN` | Kiểm tra toàn bộ input trước, ghi đơn+chi tiết nguyên khối, định nghĩa rõ “nháp” hay “chờ xác nhận” |
| P1 | Đơn khách vãng lai được phép tạo, nhưng `SalesRestController.completeOrder` đọc `dh.getKhachHang().getId()` khi tạo công nợ; `cong_no.khach_hang_id` là NOT NULL | Chốt quy tắc thanh toán/công nợ cho khách vãng lai, tránh dereference null, test từ tạo→hoàn tất→thanh toán |
| P1 | `CustomerServiceTests` và `MaterialEstimationServiceTests` dùng nhiều model/repository ở package khác nhưng thiếu import; Maven wrapper cũng chưa chạy được | Sửa baseline test Java hoặc ghi rõ test nào không thể dùng làm oracle; chỉ so sánh hành vi PHP với test nguồn đã chạy thật |
| P1 | Cart lưu trong `localStorage`; nhiều giá/role/username được JS gửi lên | Server kiểm tra lại giá, số lượng, kho và quyền; quyết định lưu cart server hay browser chỉ để hiển thị |
| P1 | Login cho phép tìm theo họ tên và có nhánh so khớp mật khẩu dạng plaintext | Chỉ đăng nhập bằng định danh duy nhất; chuyển hash BCrypt tương thích bằng `password_verify`, loại bỏ plaintext sau migration |
| P2 | 8 role nhưng 6 tài khoản mẫu và chưa có trang role giao nhận/NCC; giao diện có CSS tùy biến nhiều | Ghi rõ chức năng đã có và phần chưa có; Bootstrap 5 theo module, kiểm tra responsive/accessibility |

Các phát hiện trên là kết quả đọc mã nguồn, chưa phải báo cáo pentest hay kết luận về dữ liệu đang chạy.

## 5. Kiến trúc PHP đề xuất

```text
app/
  Http/Controllers/{Auth,Customer,Sales,Warehouse,Accounting,Management,Admin,Estimation,Ai}
  Http/Requests/       # validate input theo từng use case
  Models/              # ánh xạ 19 entity/bảng dự kiến, xác nhận với DB thực ở P0
  Policies/            # ownership và quyền trên từng bản ghi
  Services/            # transaction nghiệp vụ: Order, Stock, Return, Payment, Estimate
  Enums/               # trạng thái, vai trò, loại chứng từ
database/{migrations,seeders,factories}/
resources/views/{layouts,components,public,auth,customer,sales,warehouse,accounting,management,admin}/
resources/{css,js}/
routes/{web,api}.php
tests/{Feature,Unit}/
```

- **Xác thực:** Laravel session guard; login/logout, regenerate session, CSRF, policies/gates. Tài khoản cũ BCrypt có thể xác minh bằng PHP `password_verify` nếu hash hợp lệ; thử trên bản sao dữ liệu. Role được lưu ở DB, không tin `localStorage`.
- **API:** giữ JSON contract cho các màn fetch hiện có trong giai đoạn chuyển; dùng Form Request, API Resource, mã HTTP chuẩn và cấu trúc lỗi thống nhất.
- **Dữ liệu:** migration theo snapshot schema thực; Eloquent đặt `$table`, casts và quan hệ rõ ràng; service chịu trách nhiệm transaction. Không chạy seeder mẫu trên môi trường thật.
- **Giao diện:** Blade layout header/sidebar/footer, Bootstrap grid/table/form/modal/offcanvas/toast; tách JS khỏi template, dùng Vite hoặc bundle tương đương; nội dung Việt, trạng thái responsive và bàn phím.
- **Vận hành:** `public/` là document root, cấu hình qua `.env`, backup/restore MySQL và thư mục upload, log/health check, CI chạy PHP tests/lint. Gemini key chỉ phía server; không đưa lỗi nội bộ/secret về browser.

## 6. Kế hoạch phase có cổng nghiệm thu

Mỗi phase gồm: **đầu vào → việc làm → sản phẩm bàn giao → cổng nghiệm thu**. Chỉ chuyển phase khi cổng trước đạt; ghi checklist và commit riêng để dễ theo dõi. Ước lượng dưới đây là **độ lớn tương đối** (S/M/L), phụ thuộc tình trạng DB thật và mức độ cần giữ UI cũ.

### Phase 0 — Đóng băng hiện trạng và đặc tả (M)

- Đầu vào: mã Spring, `Database.sql`, `Migration_Material_Estimation.sql`, hai README, DB dump đã ẩn thông tin nhạy cảm nếu có, danh sách vai trò và luồng demo thực tế.
- Làm: chụp schema `SHOW CREATE TABLE`, FK/index/row count, dữ liệu mẫu; ghi 70 API + 24 mapping, request/response/status/error mẫu; sửa/chạy test nguồn nếu có thể; lập bảng quyền và trạng thái; đối chiếu README với chức năng thật, đặc biệt giá cố định hợp đồng/hạn mức nợ; chốt xử lý khách vãng lai khi hoàn tất và ghi nợ; xác nhận những phần chưa hoàn chỉnh (calculator, role giao nhận/NCC, AI).
- Bàn giao: `schema-baseline`, `route-contract-matrix`, `role-permission-matrix`, `state-transition-matrix`, bộ dữ liệu kiểm thử ẩn danh.
- Nghiệm thu: mọi bảng/route có owner trong phase; nếu có DB đang chạy thì backup/restore thử được và so schema; nếu không có thì dựng DB sạch từ SQL trong repo, ghi rõ rằng di trú dữ liệu thực chưa được xác minh; các quyết định thiếu dữ liệu được ghi rõ.

### Phase 1 — Repository PHP và nền chạy (S)

- Tạo thư mục/project PHP riêng và Git local dự kiến mang tên `Construction_materials_management_system_php`; tạo Laravel 13 với PHP 8.3 sau khi xác minh Composer/extensions/MySQL; thêm `.env.example`, `.gitignore`, README, CI cơ bản, cấu hình local/test.
- Tạo layout Blade và Bootstrap 5.3, màu/chữ/component dùng chung; cài auth nền và health page.
- Nghiệm thu: cài mới từ README, chạy ứng dụng, assets build và test cơ bản pass; `git status` không có secret. Repository GitHub đã được dùng để lưu tài liệu kế hoạch; code PHP được đẩy theo từng phase hoặc khi hoàn tất tùy quy trình chốt sau này.

### Phase 2 — Schema, dữ liệu và tương thích tài khoản (L)

- Tạo migration baseline cho 19 bảng theo `Database.sql` + migration định mức + schema DB thực; hòa giải enum `HUY`/`DA_HUY`, `CON_NO`/`CHUA_THANH_TOAN`, cột khách vãng lai, computed column và FK còn thiếu; so schema migration với DB nguồn.
- Viết import/seed dữ liệu mẫu tách biệt; thử nhập dữ liệu trên DB clone; xác minh BCrypt, null, decimal, mã hàng/đơn/phiếu không trùng, upload file.
- Nghiệm thu: với DB nguồn, import bảo toàn số bản ghi/FK và đối chiếu tổng tiền/tồn trước/sau, rollback/restore được; không có DB nguồn thì kiểm tra migration trên DB sạch với fixture và để cổng di trú dữ liệu thực ở trạng thái “chờ xác minh”.

### Phase 3 — Auth, phân quyền và hồ sơ (L)

- Đăng ký, đăng nhập, đăng xuất, đổi mật khẩu, hồ sơ; middleware role, policy ownership, admin quản lý user/role/khóa/reset bằng mật khẩu tạm ngẫu nhiên hoặc liên kết đặt mật khẩu một lần.
- Chuyển các trang auth, profile, admin sang Blade + Bootstrap; bỏ tin cậy vào role/username gửi từ browser.
- Nghiệm thu: ma trận 8 role kiểm tra được 200/403; user A không đọc/sửa đơn, hồ sơ user B; CSRF, session, logout hủy session, locked account và password hash test pass; không có nút chuyển role bằng tài khoản mẫu ở bản bàn giao.

### Phase 4 — Catalog, danh mục và giỏ hàng (M)

- Danh mục, hàng hóa, giá lẻ/sỉ, ảnh upload, danh sách/chi tiết/tìm kiếm; chuyển trang công khai và sales catalog.
- Giỏ hàng Bootstrap 5; server tính giá và kiểm tra hàng đang kinh doanh, không dùng tổng tiền do client gửi.
- Nghiệm thu: CRUD role đúng, ảnh hợp lệ, lọc/tìm kiếm và mobile layout; thay giá trong request không đổi tiền trên server.

### Phase 5 — Đơn hàng và hợp đồng (L)

- Tạo đơn từ cart/khách vãng lai/nhân viên, chi tiết đơn, đặt cọc, duyệt/hủy/hoàn tất; tạo/kích hoạt/hủy hợp đồng, tính chiết khấu và giá trị hợp đồng có validation.
- Quy tắc chuyển trạng thái duy nhất trong service, transaction cho đơn+chi tiết và hợp đồng; xác định quản lý có bắt buộc duyệt trước kích hoạt hợp đồng hay không; chặn thao tác lặp.
- Nghiệm thu: tổng tiền/đặt cọc khớp nguồn; khách chỉ thấy đơn mình; trạng thái sai trả lỗi; hợp đồng không tạo trùng cho cùng đơn nếu nghiệp vụ yêu cầu một hợp đồng.

### Phase 6 — Kho, phiếu và giao nhận (L)

- Kho/tồn, phiếu nhập/xuất/điều chỉnh, duyệt phiếu, chọn đơn để xuất, tạo giao nhận, xác nhận bàn giao và báo sự cố.
- Khóa bản ghi tồn khi cập nhật; ghi ledger biến động kho hoặc ít nhất một lịch sử nhất quán; chọn duyệt phiếu **hoặc** bàn giao là thời điểm trừ tồn và loại bỏ nhánh trừ còn lại.
- Nghiệm thu: không âm tồn khi cấm xuất vượt; duyệt/giao lặp không biến động thêm; tổng tồn và phiếu đối chiếu được; rollback nguyên khối khi lỗi giữa chừng.

### Phase 7 — Đổi trả, công nợ và thanh toán (L)

- Yêu cầu/duyệt/từ chối đổi trả; nợ phát sinh từ đơn/hợp đồng theo quy tắc đã chốt; thu tiền, đối soát thanh toán, NCC.
- Hợp nhất hai đường duyệt đổi trả vào một service; lấy giá hoàn trả theo quy tắc đã duyệt (ưu tiên giá chi tiết đơn gốc khi trả hàng), thay đổi tồn và nợ trong cùng transaction; khóa nợ khi thu tiền đồng thời; chặn khoản thu không dương/vượt dư nợ.
- Nghiệm thu: không duyệt hai lần; tiền đã trả ≤ nợ hợp lệ; người thu lấy từ session; đổi trả không làm âm nợ hoặc cộng kho sai; kiểm thử concurrency và quyền role pass.

### Phase 8 — Báo cáo và dashboard (M)

- Doanh thu, dòng tiền, tổng hợp nợ, tồn kho, thống kê quản lý; tạo/lưu/duyệt báo cáo; chuyển trang kế toán/quản lý Bootstrap.
- Thống nhất định nghĩa KPI và khoảng ngày, timezone, dữ liệu hủy/đổi trả/đặt cọc; sửa luồng lưu báo cáo để `CHO_DUYET` thực sự đi qua quản lý nếu quy trình yêu cầu; thêm export/print nếu hiện trạng cần.
- Nghiệm thu: số liệu đối chiếu với fixture và truy vấn DB; role chỉ xem phần được phép; ngày rỗng/biên được xử lý.

### Phase 9 — Tính vật liệu và trợ lý AI (M)

- Chuyển 15 định mức, 3 kiểu công trình, quy tắc làm tròn và ánh xạ 5 nhóm vật liệu; tạo lại trang calculator đang thiếu; tạo đơn từ kết quả hợp lệ theo trạng thái đã chốt ở P0.
- Chuyển chat Gemini sang HTTP client PHP; env key, timeout, rate limit, lỗi thân thiện; không lộ key hay thông tin exception nội bộ.
- Nghiệm thu: golden tests khối lượng/giá/rounding/IDOR; khách chưa login không nhận giá; chọn sai nhóm bị chặn; AI fallback ổn khi không có key.

### Phase 10 — Hoàn thiện giao diện và kiểm thử hồi quy (L)

- Rà 19 trang hiện có + calculator mới: desktop/mobile, table/form/modal, thông báo, validation, loading/empty/error, keyboard/accessibility, nội dung tiếng Việt.
- Test Feature/Unit, kiểm thử đầu-cuối các luồng role; đối chiếu contract route cũ→mới; kiểm tra upload, XSS, CSRF, IDOR, race condition, hiệu năng truy vấn và backup/restore.
- Nghiệm thu: tất cả luồng ưu tiên P0/P1 pass, không còn role giả làm nguồn quyền, QA checklist ký nhận, không có secret trong bản phát hành.

### Phase 11 — Bàn giao và cập nhật repository PHP (M)

- Viết README cài đặt, sơ đồ DB, tài khoản dev an toàn, hướng dẫn migrate/seed/backup/restore/deploy; CI tự chạy khi push; version tag đầu tiên.
- Kiểm tra lịch sử và working tree để không lộ key, dump, upload nhạy cảm; đẩy mã PHP hoàn chỉnh vào repo `Construction_materials_management_system_php` đã chứa kế hoạch, cấu hình branch protection phù hợp; chạy kiểm tra sau push và bản deploy nếu có môi trường đích.
- Nghiệm thu: clone mới và làm theo README chạy được; tests/CI pass; repo có đủ source, migration, assets build instructions và tài liệu; demo theo ma trận role hoạt động.

## 7. Thứ tự phụ thuộc và theo dõi

```text
P0 đặc tả → P1 nền → P2 dữ liệu → P3 auth
                                  ├→ P4 catalog → P5 đơn/hợp đồng → P6 kho/giao nhận
                                  │                         └────────→ P7 đổi trả/công nợ
                                  └→ P9 tính vật liệu/AI (sau P4/P5 cho đặt đơn)
P6 + P7 → P8 báo cáo → P10 QA tổng thể → P11 bàn giao/repo
```

Theo dõi mỗi phase bằng issue/milestone cùng mẫu: `ID`, `phạm vi`, `route/bảng/trang liên quan`, `đầu vào`, `công việc`, `test`, `chứng cứ`, `trạng thái` (`Chưa làm/Đang làm/Chờ xác minh/Hoàn tất`), `rủi ro còn lại`. Mỗi PR nhỏ gắn một hoặc vài issue; không gộp chuyển DB, phân quyền và UI của toàn hệ thống trong một PR.

## 8. Các lượt kiểm tra chéo kế hoạch

1. **Lượt 1 — Cấu trúc và phạm vi:** Kiểm kê Git root: 113 file theo dõi, trong đó có 3 file cấp cha từng bị bỏ sót. Đếm trực tiếp source: 19 entity ↔ 19 repository; 70 REST endpoint + 24 mapping view controller; 22 template ↔ 19 trang + 3 fragment. Phát hiện route calculator thiếu template và role seed chưa có trang tương ứng.
2. **Lượt 2 — Luồng và dữ liệu:** Đối chiếu `Database.sql` (18 bảng), script định mức (bảng thứ 19), entity, controller và seeder; ghi rõ mismatch status, cột/FK, đường đi đơn→giao nhận/kho→nợ/thanh toán và đổi trả→nợ+tồn. Gắn transaction, idempotency, decimal và kiểm thử đối soát vào P2/P5/P6/P7/P9.
3. **Lượt 3 — Đặc tả, quyền và giao diện:** Đối chiếu hai README với code, `SecurityConfig`, login/profile API, `localStorage`, template/fetch và `application.properties`. Phân biệt chức năng README hứa hẹn với chức năng đã triển khai; gắn session/role/ownership/CSRF/secret handling vào P0/P3/P10/P11; ghi rõ Bootstrap 5 là **giao diện mới**, vì code hiện tại chưa import Bootstrap.
4. **Lượt 4 — Truy vết hai chiều:** 20 tên view do controller trả về ↔ 19 trang HTML có thật + 1 calculator thiếu; 70 annotation REST ↔ 64 path API duy nhất được template/JS gọi, 5 annotation chưa có lời gọi UI và 1 cặp GET/POST chung path. So cột SQL↔entity cho 18 bảng: `cong_no.so_tien_con_lai`, `hang_hoa.ngay_cap_nhat`, `hop_dong.ngay_cap_nhat` chỉ có trong SQL; hai cột khách vãng lai chỉ có ở entity và migration bổ sung. Kiểm tra test source cho thấy import thiếu, vì vậy chưa thể coi test cũ là chuẩn đối chiếu đã chạy.
5. **Lượt 5 — Tiền và tồn kho:** Theo luồng duyệt phiếu xuất→bàn giao tìm thấy hai lần trừ tồn; theo luồng đổi trả thấy cả hai đường duyệt dùng giá hiện tại; theo luồng thu tiền thấy số âm/0 và người thu từ request; theo luồng khách vãng lai→hoàn tất thấy nguy cơ null. Từng trường hợp đã được đưa vào bảng rủi ro và cổng nghiệm thu P5–P7.
6. **Lượt 6 — Xác thực, trạng thái và khả năng chạy test:** Đọc bộ chọn role ở header, logout, tạo báo cáo/hợp đồng, cách sinh mã và ba kênh đặt hàng. Bổ sung rủi ro tài khoản demo ở browser, logout không hủy session, báo cáo tự duyệt, giá sỉ/lẻ không nhất quán và mã có nguy cơ va chạm. Thử wrapper bằng PowerShell và `cmd.exe`: cả hai dừng trước Maven; chưa có kết quả test nguồn.

**Chưa thể xác minh bằng mã nguồn:** schema/dữ liệu MySQL đang chạy, toàn bộ response thực tế, kết quả test Maven, khả năng deploy PHP trên host đích, phạm vi nghiệp vụ bổ sung của `NV_GIAO_NHAN`/`NHA_CUNG_CAP`. Đây là đầu vào bắt buộc của P0/P1, không nên coi là đã hoàn thành.

## 9. Ghi chú cho lần làm việc tiếp theo

Khi bắt đầu Phase 0, đọc tài liệu này trước, kiểm tra commit/nhánh nguồn đã thay đổi chưa rồi cập nhật số liệu. Xin DB dump đã ẩn dữ liệu nhạy cảm hoặc quyền đọc schema nếu có; hỏi người dùng về host triển khai và quy tắc nghiệp vụ còn mơ hồ sau khi đã thu thập bằng chứng. Trước khi đưa code PHP lên GitHub, kiểm tra key hiện tại trong Git history và xoay vòng nếu là key thật. Không đưa mật khẩu hay API key vào tài liệu, issue, log hoặc `.env.example`.

## 10. Phụ lục: kiểm kê REST endpoint hiện tại

Các path dưới đây lấy từ annotation controller, là danh sách để lập hợp đồng API ở Phase 0. Trong từng dòng, mỗi suffix nối với prefix ở cột đầu tiên. Tổng cộng **70 annotation REST**; `TemplateViewController` có thêm **24 annotation** (hai annotation gắn nhiều alias URL).

| Prefix | GET | POST |
|---|---|---|
| `/api/khach_hang` | `/products`, `/profile`, `/orders`, `/returns` | `/register`, `/login`, `/profile/update`, `/change-password`, `/orders/create`, `/orders/cancel`, `/orders/return`, `/orders/create-cart` |
| `/api/sales` | `/orders`, `/customers`, `/returns`, `/contracts`, `/orders/approved-no-contract` | `/orders/approve`, `/orders/cancel`, `/orders/complete`, `/orders/create`, `/returns/approve`, `/returns/reject`, `/contracts/create`, `/contracts/activate`, `/contracts/cancel` |
| `/api/warehouse` | `/summary`, `/stock`, `/slips`, `/form-data`, `/orders-to-export`, `/deliveries` | `/slips/create`, `/stock/adjust`, `/delivery/create`, `/delivery/confirm-handover`, `/delivery/report-issue` |
| `/api/accounting` | `/debts/customers`, `/summary`, `/payments`, `/suppliers`, `/reports/revenue`, `/reports/cashflow`, `/reports/debt-summary`, `/contracts` | `/debts/collect`, `/reports/save`, `/suppliers` |
| `/api/management` | `/summary`, `/orders`, `/pending-approvals`, `/inventory-report` | `/contracts/approve`, `/returns/approve`, `/returns/reject`, `/reports/approve`, `/inventory-slips/approve` |
| `/api/admin` | `/users`, `/summary`, `/roles` | `/users/create`, `/users/toggle-lock`, `/users/reset-password`, `/users/change-role`, `/users/update`, `/users/delete` |
| `/api/vat-lieu` | `/danh-sach-san-pham` | `/tinh-toan`, `/dat-hang-nhap` |
| `/api/ai` | — | `/chat` |

`TemplateViewController` còn trả các trang công khai (`/`, `/home`, `/san_pham`, `/san_pham/chi_tiet`, `/login`, `/register`, `/auth`), trang hồ sơ/calculator, trang khách hàng, kinh doanh, kế toán, quản lý, kho, admin và 4 POST thêm/xóa hàng hóa hoặc danh mục. Danh sách đầy đủ các trang nằm trong `ht_vlxd/src/main/resources/templates/`; route calculator hiện thiếu file tương ứng.

Kiểm tra ngược template/JS tìm được **64 path `/api/...` duy nhất**; chúng tương ứng với route backend. **Năm annotation REST** không có lời gọi trực tiếp trong template/JS: ba endpoint `/api/vat-lieu/*` (vì thiếu calculator view), `/api/admin/roles` và `/api/accounting/contracts`. Chênh lệch còn lại giữa 70 annotation và 64 path là do GET/POST `/api/accounting/suppliers` dùng chung một path. Khi chuyển PHP, không xóa endpoint chỉ vì UI hiện chưa gọi; quyết định giữ/bỏ sau khi xác nhận API có người dùng khác hay không.

## 11. Cách kiểm tra lại số liệu khảo sát

Chạy tại `ht_vlxd/` trên đúng commit đã ghi ở đầu tài liệu:

```powershell
git -C .. ls-files | Measure-Object                   # 113 file Git theo dõi
rg '@Entity' src/main/java/com/example/ht_vlxd/Model | Measure-Object  # 19 entity
Select-String -Path ../Database.sql -Pattern '^CREATE TABLE' | Measure-Object # 18 bảng SQL gốc
Get-ChildItem src/main/resources/templates -Recurse -Filter *.html | Measure-Object # 22 template
```

Đếm route phải loại `TemplateViewController` để được 70 annotation REST; controller này riêng có 24 annotation. Kiểm tra trực tiếp `mvnw.cmd` và test Java trước khi dùng số liệu “test pass”, vì hiện wrapper chưa chạy được. Kết quả đếm là dấu mốc của commit khảo sát, phải chạy lại nếu code nguồn đổi.
