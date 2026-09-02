# Bài 4: Cài Tutor LMS, quan sát thay đổi hệ thống và tạo Course

## Mục tiêu

Sau bài này, bạn có thể:

- Phân biệt việc cài plugin với kích hoạt plugin.
- Nhận biết thay đổi trên wp-admin và frontend sau khi kích hoạt Tutor LMS.
- Xác định các bảng, option, Page, role và capability Tutor tạo ra.
- Hiểu luồng activation của Tutor LMS.
- Biết Course là custom post type `courses`.
- Quan sát Course và metadata trong database.
- Tự kiểm tra trạng thái Tutor LMS bằng WP-CLI và SQL chỉ đọc.

## 1. Trạng thái trước khi cài

Trước Bài 4, `onthi-lab` chỉ có hai plugin mặc định:

```text
akismet   inactive
hello     inactive
```

Database chưa có:

- Bảng mang tiền tố `wp_tutor_`.
- Option bắt đầu bằng `tutor`.
- Post type `courses`, `lesson`, `topics` hoặc `tutor_enrolled`.
- Page hệ thống Tutor LMS.

## 2. Plugin đã cài

Thông tin sau khi cài và kích hoạt:

| Thuộc tính | Giá trị |
| --- | --- |
| Plugin slug | `tutor` |
| Tên | Tutor LMS |
| Tác giả | Themeum |
| Phiên bản | `4.0.7` |
| Trạng thái | `active` |

Tutor LMS được cài tại:

```text
D:\xampp-new\htdocs\onthi-lab\wp-content\plugins\tutor
```

Không chỉnh sửa trực tiếp source plugin. Khi plugin cập nhật, toàn bộ thư mục này có thể bị thay thế.

## 3. Giao diện thay đổi như thế nào?

### 3.1. Thay đổi trong wp-admin

Tutor LMS đăng ký một menu cấp cao mới tên **Tutor LMS** ở thanh menu bên trái.

Tùy quyền user và cấu hình, các submenu có thể gồm:

```text
Tutor LMS
├── Courses
├── Create Course
├── Categories
├── Tags
├── Students
├── Announcements
├── Quiz Attempts
├── Q&A
├── Instructors
├── Withdraw Requests
├── Addons
├── Tools
├── Settings
└── Upgrade to Pro
```

Danh sách này được đăng ký bằng PHP trong:

```text
wp-content/plugins/tutor/classes/Admin.php
```

Điểm quan trọng: menu wp-admin không phải là các dòng `page` trong `wp_posts`. Plugin tạo menu bằng các hàm WordPress như `add_menu_page()` và `add_submenu_page()` mỗi khi trang quản trị được tải.

Khi Tutor bị vô hiệu hóa, menu biến mất dù các bảng và Page có thể vẫn còn trong database.

### 3.2. Thay đổi trên frontend

Tutor tạo các Page hệ thống và dùng template, shortcode hoặc routing để render giao diện LMS.

Các Page đã tạo:

| ID | Tiêu đề | Slug | Nội dung lưu trong `post_content` |
| ---: | --- | --- | --- |
| 10 | Dashboard | `dashboard` | Trống |
| 11 | Student Registration | `student-registration` | `[tutor_student_registration_form]` |
| 12 | Instructor Registration | `instructor-registration` | `[tutor_instructor_registration_form]` |
| 13 | Cart | `cart` | Trống |
| 14 | Checkout | `checkout` | Trống |

Page có nội dung trống không có nghĩa là frontend trống. Tutor nhận diện Page ID hoặc URL rồi render template của plugin.

Frontend không nhất thiết thay đổi toàn bộ ngay sau khi cài. Trang chủ và các Page WordPress cũ vẫn do theme hiện tại render. Giao diện Tutor xuất hiện rõ khi truy cập:

- Course archive hoặc Course detail.
- Tutor Dashboard.
- Form đăng ký Student/Instructor.
- Cart và Checkout của Tutor native eCommerce.

Tutor cũng có thể thêm block, stylesheet, script và các route riêng khi cần.

### 3.3. Cách tự quan sát

Trong wp-admin:

1. Nhìn thanh menu bên trái và tìm **Tutor LMS**.
2. Vào **Trang → Tất cả các trang**.
3. Tìm Dashboard, Student Registration, Instructor Registration, Cart và Checkout.
4. Mở Student Registration để thấy shortcode trong nội dung.
5. Dùng nút **Xem** để quan sát giao diện mà Tutor render trên frontend.

Không chỉnh sửa hoặc xóa các Page hệ thống trong lúc học.

## 4. Các bảng Tutor tạo ra

Tutor LMS 4.0.7 đã tạo 20 bảng có tiền tố `wp_tutor_`:

```text
wp_tutor_carts
wp_tutor_cart_items
wp_tutor_coupons
wp_tutor_coupon_applications
wp_tutor_coupon_usages
wp_tutor_customers
wp_tutor_earnings
wp_tutor_legal_consents
wp_tutor_legal_consent_logs
wp_tutor_ordermeta
wp_tutor_orders
wp_tutor_order_itemmeta
wp_tutor_order_items
wp_tutor_quiz_attempts
wp_tutor_quiz_attempt_answers
wp_tutor_quiz_questions
wp_tutor_quiz_question_answers
wp_tutor_scheduler
wp_tutor_user_consents
wp_tutor_withdraws
```

Các nhóm dữ liệu chính:

- Native eCommerce: cart, order, coupon và customer.
- Quiz: attempt, question và answer.
- Earnings và withdraw.
- Scheduler.
- Legal consent và GDPR.

Các bảng eCommerce trên thuộc hệ thống bán hàng native của Tutor 4.x. Sự tồn tại của chúng không có nghĩa WooCommerce đã được cài.

Course, Topic, Lesson và Enrollment không nhất thiết nằm trong những bảng riêng này. Tutor tận dụng `wp_posts` và `wp_postmeta` cho nhiều đối tượng LMS.

## 5. Option Tutor LMS

Một số option đã được tạo:

```text
tutor_option
tutor_version
tutor_first_activation_time
tutor_wizard
tutor_gdpr_db_schema_version
tutor_batch_processor_quiz_attempt_migrator
```

### `tutor_option`

Tên chính xác là `tutor_option`, không có dấu gạch chéo.

Đây là một dòng trong `wp_options` chứa nhiều cấu hình Tutor dưới dạng PHP serialized array:

```text
tutor_option
├── pagination_per_page
├── courses_per_page
├── course_permalink_base
├── lesson_permalink_base
├── monetize_by
├── currency_code
├── tutor_dashboard_page_id
├── student_register_page
├── instructor_register_page
├── tutor_cart_page_id
└── tutor_checkout_page_id
```

Không chỉnh `option_value` thủ công trong phpMyAdmin. Khi cần thay một trường con, dùng giao diện Tutor LMS hoặc API hỗ trợ serialized data.

Option này còn chứa email gửi đi được Tutor sao chép từ cấu hình WordPress trong lần kích hoạt. Môi trường hiện vẫn chứa email thật theo quyết định giữ nguyên của người học. Không hiển thị, chia sẻ hoặc sử dụng địa chỉ đó trong bài thực hành.

## 6. Role và capability

Tutor bổ sung role/capability cho hệ thống.

Admin ID `1` được quan sát có hai role:

```text
administrator
tutor_instructor
```

Tutor cũng tạo các user metadata:

```text
_is_tutor_instructor
_tutor_instructor_status
_tutor_instructor_approved
```

User `hocvien_bai2` vẫn chỉ có role `subscriber`.

Role definitions được lưu trong option `wp_user_roles`; role gán cho từng user vẫn nằm trong `wp_usermeta` qua `wp_capabilities`.

## 7. Luồng kích hoạt Tutor LMS

Activation hook được đăng ký tại:

```text
wp-content/plugins/tutor/tutor.php
```

Callback chính:

```php
Tutor::tutor_activate()
```

Luồng khái quát:

```text
Kích hoạt plugin
    -> đặt cờ cập nhật permalink
    -> tạo hoặc cập nhật database schema
    -> lưu tutor_option mặc định
    -> tạo role và capability
    -> tạo Page hệ thống
    -> lưu tutor_version
    -> đăng ký scheduled event
```

Các method quan trọng trong `wp-content/plugins/tutor/classes/Tutor.php`:

| Method | Vai trò |
| --- | --- |
| `Tutor::tutor_activate()` | Điều phối activation |
| `Tutor::create_database()` | Tạo/cập nhật bảng Tutor |
| `Tutor::manage_tutor_roles_and_permissions()` | Tạo role và capability |
| `Tutor::save_data()` | Tạo các Page ban đầu |
| `Tutor::default_options()` | Khai báo cấu hình mặc định |

Cart và Checkout được quản lý bởi:

```text
wp-content/plugins/tutor/ecommerce/CartController.php
wp-content/plugins/tutor/ecommerce/CheckoutController.php
```

WordPress API tham gia gồm:

```php
register_activation_hook()
dbDelta()
update_option()
wp_insert_post()
add_role()
add_cap()
wp_schedule_event()
```

Tutor đăng ký scheduled event:

```text
tutor_once_in_day_run_schedule
```

## 8. Course là custom post type

Tutor đăng ký Course với post type:

```text
courses
```

Post type này được cấu hình trong:

```text
wp-content/plugins/tutor/classes/Config.php
```

và đăng ký trong:

```text
wp-content/plugins/tutor/classes/Post_types.php
```

Tutor đăng ký post type ở hook WordPress:

```php
init
```

Điều này nối trực tiếp với Bài 1: Course vẫn sử dụng kiến trúc `wp_posts` + `wp_postmeta`, chỉ khác `post_type`.

## 9. Course thực hành

Course đã tạo:

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `16` |
| `post_title` | `Khóa học WordPress thực hành` |
| `post_name` | `khoa-hoc-wordpress-thuc-hanh` |
| `post_status` | `draft` |
| `post_type` | `courses` |
| `post_author` | `1` |
| `post_parent` | `0` |

Description nằm trong `wp_posts.post_content`:

```html
<p>Khóa học mẫu dùng để quan sát cách Tutor LMS lưu Course, Topic và Lesson trong database.</p>
```

Metadata được quan sát:

| `meta_key` | Ý nghĩa/giá trị |
| --- | --- |
| `_tutor_course_price_type` | `free` |
| `_tutor_course_settings` | Mảng serialized chứa thiết lập enrollment |
| `_video` | Cấu hình video, hiện là mảng trống |
| `_tutor_enable_qa` | `no` |
| `_tutor_is_public_course` | `no` |
| `tutor_course_sale_price` | `0` |
| `_course_duration` | Thời lượng khóa học |
| `_tutor_course_level` | `intermediate` |

Chưa có Topic, Lesson, taxonomy hoặc child post nào liên kết với Course ID `16`.

Một Page `auto-draft` ID `15` cũng tồn tại nhưng không liên kết với Course: `post_type = page`, `post_parent = 0`. Không xóa trong bài này.

## 10. Tự kiểm tra bằng WP-CLI

Kiểm tra plugin:

```powershell
cd D:\xampp-new\htdocs\onthi-lab

& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" plugin get tutor --fields=name,status,version,title,author
```

Liệt kê Course:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" post list --post_type=courses --fields=ID,post_title,post_name,post_status,post_author
```

Các lệnh này chỉ đọc dữ liệu.

## 11. Tự kiểm tra bằng SQL

### Kiểm tra bảng Tutor

```sql
SELECT TABLE_NAME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'onthi_lab'
  AND TABLE_NAME LIKE 'wp_tutor_%'
ORDER BY TABLE_NAME;
```

### Kiểm tra Page hệ thống

```sql
SELECT ID, post_title, post_name, post_status, post_content
FROM wp_posts
WHERE ID BETWEEN 10 AND 14
ORDER BY ID;
```

### Kiểm tra các option Tutor mà không đọc giá trị

```sql
SELECT option_id, option_name, autoload, LENGTH(option_value) AS value_bytes
FROM wp_options
WHERE option_name LIKE 'tutor%'
ORDER BY option_id;
```

### Kiểm tra Course

```sql
SELECT
    ID,
    post_author,
    post_title,
    post_name,
    post_status,
    post_type,
    post_parent
FROM wp_posts
WHERE ID = 16;
```

### Kiểm tra Course metadata

```sql
SELECT meta_id, post_id, meta_key, meta_value
FROM wp_postmeta
WHERE post_id = 16
ORDER BY meta_id;
```

Không xuất hoặc chia sẻ toàn bộ giá trị `tutor_option`, vì nó có thể chứa thông tin cấu hình riêng.

## 12. Điều cần ghi nhớ

1. Cài plugin là đưa source vào `wp-content/plugins`; kích hoạt mới chạy activation logic.
2. Menu Tutor LMS trong wp-admin được đăng ký bằng code, không phải Page.
3. Page hệ thống là dữ liệu trong `wp_posts` với `post_type = page`.
4. Page có `post_content` trống vẫn có thể được Tutor render bằng template.
5. Tutor tạo bảng riêng cho quiz, eCommerce, earnings, scheduler và consent.
6. Course vẫn nằm trong `wp_posts`, với `post_type = courses`.
7. Thiết lập Course nằm trong `wp_postmeta`.
8. `tutor_option` là một option dạng serialized array chứa nhiều thiết lập.
9. Role và capability Tutor được tích hợp vào hệ thống phân quyền WordPress.
10. Không sửa WordPress core hoặc source Tutor LMS trực tiếp.

## 13. Câu hỏi ôn tập

1. Menu Tutor LMS trong wp-admin có được lưu như một Page trong `wp_posts` không?
2. Vì sao Page Dashboard có `post_content` trống nhưng vẫn có thể hiển thị giao diện Tutor?
3. Sự tồn tại của `wp_tutor_orders` có chứng minh WooCommerce đã được cài không?
4. Course ID `16` được phân biệt với Page bằng cột nào?
5. Thiết lập giá và thời lượng Course nằm trong bảng nào?
6. `tutor_option` khác các option `tutor_version` và `tutor_wizard` như thế nào?
7. Khi vô hiệu hóa Tutor LMS, vì sao menu biến mất nhưng dữ liệu có thể vẫn còn?
8. File nào đăng ký admin menu và file nào đăng ký custom post type?

## 14. Chuẩn bị cho Bài 5

Bài 5 sẽ xây cấu trúc:

```text
Course ID 16
└── Topic
    └── Lesson
```

Sau đó truy vấn `wp_posts` để quan sát `post_type`, `post_parent` và thứ tự nội dung.
