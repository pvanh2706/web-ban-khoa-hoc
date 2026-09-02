# Bài 6: Enrollment và trạng thái enrollment

## Mục tiêu

- Hiểu Tutor LMS lưu một lượt đăng ký Course như thế nào.
- Đọc được quan hệ giữa enrollment, học viên và Course.
- Hiểu vì sao trạng thái `completed` cho phép học, còn `cancel` không được xem là enrollment hợp lệ.
- Biết class, method và hook tham gia quá trình tạo hoặc đổi trạng thái enrollment.
- Biết cách tự kiểm tra bằng giao diện, SQL, source và WP-CLI.

## 1. Dữ liệu thực hành

Học viên `hocvien_bai2` đã đăng ký Course miễn phí **Khóa học WordPress thực hành**.

Tutor LMS tạo bản ghi:

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `21` |
| `post_type` | `tutor_enrolled` |
| `post_status` ban đầu | `completed` |
| `post_author` | `2` — user `hocvien_bai2` |
| `post_parent` | `16` — Course được đăng ký |
| `post_date` | `2026-09-01 16:01:30` |

Quan hệ có thể đọc như sau:

```text
wp_users.ID = 2
       ↑
       │ post_author
Enrollment ID 21
       │ post_parent
       ↓
Course ID 16
```

- `post_author` trả lời: ai đăng ký?
- `post_parent` trả lời: đăng ký Course nào?
- `post_status` trả lời: enrollment đang ở trạng thái nào?

Enrollment được lưu trong bảng chuẩn `wp_posts`, không có bảng enrollment riêng của Tutor LMS.

## 2. Truy vấn enrollment

```sql
SELECT
    enrollment.ID AS enrollment_id,
    enrollment.post_type,
    enrollment.post_status,
    enrollment.post_author AS user_id,
    user.user_login,
    enrollment.post_parent AS course_id,
    course.post_title AS course_title,
    enrollment.post_date
FROM wp_posts AS enrollment
LEFT JOIN wp_users AS user
    ON user.ID = enrollment.post_author
LEFT JOIN wp_posts AS course
    ON course.ID = enrollment.post_parent
WHERE enrollment.post_type = 'tutor_enrolled';
```

Kết quả thực hành:

```text
21 | tutor_enrolled | completed | 2 | hocvien_bai2 | 16 | Khóa học WordPress thực hành | 2026-09-01 16:01:30
```

## 3. Metadata của enrollment miễn phí

Truy vấn:

```sql
SELECT meta_id, post_id, meta_key, meta_value
FROM wp_postmeta
WHERE post_id = 21
ORDER BY meta_id;
```

Kết quả trống. Đây là Course miễn phí nên enrollment này chưa liên kết với Order hoặc Product.

Khi enrollment được tạo từ một đơn hàng, Tutor có thể dùng các meta key:

```text
_tutor_enrolled_by_order_id
_tutor_enrolled_by_product_id
```

## 4. Source tạo enrollment

Class chính:

```php
Tutor\Models\EnrollmentModel
```

File:

```text
wp-content/plugins/tutor/models/EnrollmentModel.php
```

Method tạo enrollment:

```php
EnrollmentModel::do_enroll()
```

Luồng xử lý khái quát:

```text
Kiểm tra Course có thể truy cập
→ lọc quyền đăng ký
→ kiểm tra enrollment đã tồn tại
→ xác định trạng thái
→ wp_insert_post()
→ cập nhật user meta đánh dấu học viên
→ lưu Order/Product metadata nếu có order_id
→ chạy hook sau đăng ký
```

Với Course miễn phí, trạng thái mặc định là:

```php
$enrolment_status = EnrollmentModel::STATUS_COMPLETED;
```

Nếu Course yêu cầu mua, Tutor đổi trạng thái ban đầu thành `pending`.

## 5. Kiểm tra quyền học

Method:

```php
EnrollmentModel::is_enrolled()
```

Theo mặc định, truy vấn của method này yêu cầu:

```sql
post_status = 'completed'
```

Kết quả quan sát trước khi đổi trạng thái:

- `hocvien_bai2` có enrollment `completed`.
- Học viên mở được Course.
- Học viên vào được Lesson qua nút bắt đầu hoặc tiếp tục học.

Điều này cho thấy việc Lesson có `post_status = publish` chưa đủ để cấp quyền. Tutor còn kiểm tra enrollment của user với Course.

## 6. Hook liên quan

Trong quá trình tạo enrollment:

```php
tutor_allow_course_enrollment
tutor_before_enroll
tutor_enroll_data
tutor_after_enroll
tutor_after_enrolled
```

- `tutor_allow_course_enrollment`: filter cho phép hoặc từ chối đăng ký.
- `tutor_enroll_data`: filter dữ liệu trước khi `wp_insert_post()`.
- `tutor_after_enroll`: action chạy với enrollment mới ở trạng thái `pending` hoặc `completed`.
- `tutor_after_enrolled`: action chỉ chạy khi enrollment mới có trạng thái `completed`.

Khi dùng `EnrollmentModel::update_enrollments()`, Tutor chạy hook động:

```php
tutor_enrollment/after/{status}
```

Ví dụ khi chuyển sang `cancel`:

```php
tutor_enrollment/after/cancel
```

## 7. Thí nghiệm đổi trạng thái

### Kiểm tra đúng database trước khi thay đổi

```powershell
cd D:\xampp-new\htdocs\onthi-lab

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  config get DB_NAME
```

Chỉ tiếp tục nếu kết quả là:

```text
onthi_lab
```

### Chuyển enrollment 21 sang `cancel`

```powershell
& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  eval "var_export( \Tutor\Models\EnrollmentModel::update_enrollments( 'cancel', array( 21 ) ) );"
```

Lệnh dùng method của Tutor LMS thay vì cập nhật SQL trực tiếp, nhờ đó hook trạng thái vẫn được chạy.

Kết quả ngày 01-09-2026:

```text
true
```

Sau lệnh, bản ghi là:

```text
ID=21 | post_type=tutor_enrolled | post_status=cancel | post_author=2 | post_parent=16
```

### Khôi phục về `completed`

Chỉ chạy sau khi đã quan sát tác động của trạng thái `cancel`:

```powershell
& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  eval "var_export( \Tutor\Models\EnrollmentModel::update_enrollments( 'completed', array( 21 ) ) );"
```

Hook tương ứng khi khôi phục:

```php
tutor_enrollment/after/completed
```

Kết quả khôi phục:

```text
true
```

Sau khi kiểm tra lại trong database:

```text
ID=21 | post_type=tutor_enrolled | post_status=completed | post_author=2 | post_parent=16
```

## 8. Kết quả quan sát trên giao diện

Khi enrollment ID `21` có trạng thái `cancel`, học viên không còn xem được nội dung Lesson. Tutor LMS hiển thị thông báo yêu cầu đăng ký Course để xem nội dung.

Điều này chứng minh:

```text
Lesson publish
       +
Enrollment không phải completed
       ↓
Tutor LMS vẫn chặn nội dung Lesson
```

Sau khi quan sát, enrollment được khôi phục về `completed` bằng `EnrollmentModel::update_enrollments()`.

## 9. Trạng thái hiện tại của bài thực hành

Enrollment ID `21` hiện đã trở lại trạng thái `completed`. Học viên có thể tiếp tục truy cập Course và Lesson.

## 10. Ý nghĩa các trạng thái chính

| Trạng thái | Ý nghĩa trong bài học |
| --- | --- |
| `completed` | Enrollment có hiệu lực; `is_enrolled()` mặc định công nhận và học viên được truy cập nội dung. |
| `pending` | Enrollment đang chờ hoàn tất điều kiện, thường liên quan đến Course cần mua; chưa được công nhận như `completed`. |
| `cancel` | Enrollment đã bị hủy; học viên không còn quyền truy cập nội dung Course. |

`completed` ở đây là trạng thái **của enrollment**, không có nghĩa học viên đã học xong Course. Tiến độ và việc hoàn thành Course là dữ liệu khác.

Trong `do_enroll()`:

- Course miễn phí mặc định sinh enrollment `completed`.
- Course cần mua mặc định sinh enrollment `pending`.
- Sau này, hệ thống thanh toán có thể chuyển enrollment sang `completed` khi điều kiện thanh toán được đáp ứng.

## 11. Đã học được gì?

1. Tutor LMS lưu enrollment bằng custom post type `tutor_enrolled` trong `wp_posts`.
2. `post_author` liên kết enrollment với user.
3. `post_parent` liên kết enrollment với Course.
4. `post_status` quyết định enrollment có hiệu lực hay không.
5. Enrollment của Course miễn phí không nhất thiết có dòng trong `wp_postmeta`.
6. `EnrollmentModel::do_enroll()` tạo enrollment.
7. `EnrollmentModel::is_enrolled()` kiểm tra quyền và mặc định chỉ nhận trạng thái `completed`.
8. `EnrollmentModel::update_enrollments()` thay đổi trạng thái và phát hook tương ứng.
9. Trạng thái `publish` của Lesson không tự cấp quyền học nếu Course yêu cầu enrollment.
10. Nên dùng API/method của Tutor thay vì tự cập nhật SQL để các hook liên quan được chạy.

## 12. Dữ liệu được lưu ở đâu?

| Dữ liệu | Vị trí |
| --- | --- |
| Bản ghi enrollment | `wp_posts` |
| Học viên của enrollment | `wp_posts.post_author` |
| Course của enrollment | `wp_posts.post_parent` |
| Trạng thái enrollment | `wp_posts.post_status` |
| Liên kết Order/Product nếu có | `wp_postmeta` |
| Dấu hiệu user là học viên Tutor | `wp_usermeta`, meta key `_is_tutor_student` |

## 13. File và class xử lý

| Thành phần | Vai trò |
| --- | --- |
| `wp-content/plugins/tutor/models/EnrollmentModel.php` | Tạo, tìm, cập nhật và xóa enrollment. |
| `Tutor\Models\EnrollmentModel::do_enroll()` | Tạo enrollment mới. |
| `Tutor\Models\EnrollmentModel::is_enrolled()` | Kiểm tra user có enrollment `completed` hay không. |
| `Tutor\Models\EnrollmentModel::update_enrollments()` | Cập nhật trạng thái enrollment. |
| `wp-content/plugins/tutor/classes/Course.php` | Nhận yêu cầu đăng ký Course và gọi model enrollment. |
| `wp-content/plugins/tutor/classes/Lesson.php` | Kiểm tra quyền trước khi cho xem nội dung Lesson. |
| `wp-content/plugins/tutor/classes/Post_types.php` | Đăng ký post type enrollment với WordPress. |

Không chỉnh sửa trực tiếp các file của Tutor LMS. Nếu cần tùy biến sau này, code phải nằm trong plugin riêng và sử dụng hook/API công khai.

## 14. Cách tự kiểm tra kết quả

### Kiểm tra một enrollment bằng SQL

```sql
SELECT ID, post_type, post_status, post_author, post_parent
FROM wp_posts
WHERE ID = 21
  AND post_type = 'tutor_enrolled';
```

Trạng thái cuối mong đợi:

```text
21 | tutor_enrolled | completed | 2 | 16
```

### Kiểm tra trên giao diện

1. Đăng nhập bằng `hocvien_bai2`.
2. Mở Course **Khóa học WordPress thực hành**.
3. Đi vào Lesson từ nút bắt đầu hoặc tiếp tục học.
4. Enrollment `completed` thì Lesson mở được.
5. Enrollment `cancel` thì Tutor yêu cầu đăng ký Course.

## 15. Câu hỏi ôn tập

1. Enrollment của Tutor LMS dùng `post_type` nào?
2. Cột nào cho biết học viên của enrollment?
3. Cột nào cho biết Course được đăng ký?
4. Tại sao Course miễn phí tạo enrollment ở trạng thái `completed`?
5. Vì sao Lesson `publish` vẫn bị chặn khi enrollment là `cancel`?
6. `completed` của enrollment có đồng nghĩa học viên đã hoàn thành Course không?
7. Method nào tạo enrollment, method nào kiểm tra quyền và method nào đổi trạng thái?
8. Hook nào chạy sau khi tạo enrollment `completed`?
9. Hook động nào chạy sau khi đổi enrollment sang `cancel`?
10. Vì sao nên gọi method của Tutor thay vì cập nhật `post_status` trực tiếp bằng SQL?
