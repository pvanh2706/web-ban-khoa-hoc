# Bài 5: Quan hệ Course, Topic, Lesson và `post_parent`

## Mục tiêu

Sau bài này, bạn có thể:

- Xác định post type của Course, Topic và Lesson.
- Hiểu Tutor LMS biểu diễn cấu trúc khóa học bằng `post_parent`.
- Hiểu vai trò của `menu_order` trong Curriculum.
- Phân biệt trạng thái của Course với trạng thái của nội dung con.
- Phân biệt nút lưu Course, Topic và Lesson trong Course Builder 4.0.7.
- Biết hàm, file và hook tham gia quá trình tạo Topic và Lesson.
- Tự truy vấn toàn bộ cấu trúc Curriculum bằng SQL.

## 1. Cấu trúc đã tạo

Kết quả thực hành:

```text
Course ID 16: Khóa học WordPress thực hành
└── Topic ID 19: Chủ đề 1 - Nền tảng WordPress
    └── Lesson ID 20: Bài 1 - WordPress lưu dữ liệu ở đâu?
```

Cả ba đối tượng đều nằm trong bảng `wp_posts`.

## 2. Dữ liệu Course

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `16` |
| `post_title` | `Khóa học WordPress thực hành` |
| `post_name` | `khoa-hoc-wordpress-thuc-hanh` |
| `post_type` | `courses` |
| `post_status` | `draft` |
| `post_parent` | `0` |
| `menu_order` | `0` |
| `post_author` | `1` |

Course là đối tượng gốc nên `post_parent = 0`.

Các thiết lập mở rộng của Course như giá, cấp độ, thời lượng và enrollment nằm trong `wp_postmeta`.

## 3. Dữ liệu Topic

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `19` |
| `post_title` | `Chủ đề 1 - Nền tảng WordPress` |
| `post_name` | `chu-de-1-nen-tang-wordpress` |
| `post_type` | `topics` |
| `post_status` | `publish` |
| `post_parent` | `16` |
| `menu_order` | `1` |
| `post_author` | `1` |

Summary của Topic nằm trong `post_content`:

```text
Tìm hiểu cấu trúc dữ liệu cơ bản của WordPress trước khi xây dựng website khóa học.
```

Quan hệ:

```text
Topic.post_parent = 16 = Course.ID
```

Topic không có dòng riêng trong `wp_postmeta` ở lần thực hành này. Các dữ liệu cần thiết đều nằm trong cột chuẩn của `wp_posts`.

## 4. Dữ liệu Lesson

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `20` |
| `post_title` | `Bài 1 - WordPress lưu dữ liệu ở đâu?` |
| `post_name` | `bai-1-wordpress-luu-du-lieu-o-dau` |
| `post_type` | `lesson` |
| `post_status` | `publish` |
| `post_parent` | `19` |
| `menu_order` | `1` |
| `post_author` | `1` |

Nội dung Lesson nằm trong `post_content`:

```html
<p>WordPress lưu nội dung chính trong wp_posts và lưu dữ liệu mở rộng trong các bảng metadata tương ứng.</p>
```

Quan hệ:

```text
Lesson.post_parent = 19 = Topic.ID
```

Lesson không có metadata ở lần thực hành này vì chưa thêm video, ảnh, attachment, preview hoặc content drip.

## 5. Cách đọc toàn bộ cây quan hệ

```text
wp_posts.ID = 16
post_type    = courses
post_parent  = 0
        │
        └── wp_posts.ID = 19
            post_type    = topics
            post_parent  = 16
            menu_order   = 1
                    │
                    └── wp_posts.ID = 20
                        post_type    = lesson
                        post_parent  = 19
                        menu_order   = 1
```

Tutor không đặt `Lesson.post_parent` trực tiếp bằng Course ID. Lesson trỏ đến Topic, còn Topic trỏ đến Course.

Để tìm Course của một Lesson, cần đi hai cấp:

```text
Lesson -> Topic -> Course
```

## 6. `menu_order` dùng để làm gì?

`menu_order` biểu thị thứ tự tương đối trong Curriculum:

- Topic đầu tiên có `menu_order = 1`.
- Lesson đầu tiên trong Topic có `menu_order = 1`.
- Khi thêm Topic hoặc Lesson tiếp theo, Tutor sẽ cấp thứ tự tiếp theo.
- Khi kéo thả nội dung trong Course Builder, Tutor cập nhật thứ tự này.

Không nên tự chỉnh `menu_order` bằng SQL. Dùng thao tác kéo thả trong Course Builder để Tutor cập nhật toàn bộ quan hệ và cache liên quan.

## 7. Trạng thái Course và nội dung con

Trạng thái quan sát được:

```text
Course = draft
Topic  = publish
Lesson = publish
```

Điều này không làm Topic và Lesson tự động trở thành khóa học công khai. Tutor kiểm tra Course cha và quyền truy cập khi render nội dung.

Vì vậy, không nên kết luận quyền truy cập Lesson chỉ dựa vào `Lesson.post_status`.

## 8. Ba nút lưu có phạm vi khác nhau

Tutor LMS Course Builder 4.0.7 có ba thao tác lưu riêng:

```text
Course Save as Draft -> lưu dữ liệu Course
Topic OK            -> tạo hoặc cập nhật Topic
Lesson Save         -> tạo hoặc cập nhật Lesson
```

### Lỗi đã gặp trong thực hành

Sau khi nhấn **Add Topic**, Course Builder chỉ thêm một khối Topic tạm trong state của trình duyệt. Nếu nhập title/summary rồi nhấn **Save as Draft** của Course mà chưa nhấn **OK** trong khối Topic:

- Course được cập nhật thành công.
- Giao diện tạm thời vẫn có thể hiện Topic.
- Database chưa có dòng `post_type = topics`.
- Sau khi tải lại trang, Topic biến mất.

Topic chỉ được lưu khi:

1. Nhập title và summary.
2. Cuộn đến cuối khối Topic.
3. Nhấn nút **OK** cạnh nút Cancel.
4. Chờ thông báo `Topic saved successfully`.

Lesson cũng phải được lưu bằng nút **Save** trong cửa sổ Lesson trước khi lưu Course.

## 9. Luồng lưu Topic

Course Builder gửi AJAX action:

```text
tutor_save_topic
```

WordPress đăng ký action:

```php
wp_ajax_tutor_save_topic
```

Method xử lý:

```php
Tutor\Course::tutor_save_topic()
```

File:

```text
wp-content/plugins/tutor/classes/Course.php
```

Payload khái quát:

```text
course_id = 16
title     = Chủ đề 1 - Nền tảng WordPress
summary   = ...
```

Tutor tạo mảng post:

```php
array(
    'post_type'    => 'topics',
    'post_title'   => $topic_title,
    'post_content' => $topic_summary,
    'post_status'  => 'publish',
    'post_author'  => get_current_user_id(),
    'post_parent'  => $course_id,
    'menu_order'   => $next_topic_order_id,
)
```

Sau đó gọi:

```php
wp_insert_post()
```

Response khi tạo thành công có status code ứng dụng `201` và trả về Topic ID.

## 10. Luồng lưu Lesson

Course Builder gửi AJAX action:

```text
tutor_save_lesson
```

WordPress đăng ký:

```php
wp_ajax_tutor_save_lesson
```

Method xử lý:

```php
Tutor\Lesson::ajax_save_lesson()
```

File:

```text
wp-content/plugins/tutor/classes/Lesson.php
```

Tutor kiểm tra:

- Nonce hợp lệ.
- Có `topic_id`.
- User hiện tại được quản lý Topic.
- Dữ liệu đầu vào hợp lệ.

Mảng dữ liệu Lesson gồm:

```php
array(
    'post_type'      => 'lesson',
    'post_title'     => $title,
    'post_name'      => sanitize_title( $title ),
    'post_content'   => $post_content,
    'post_status'    => 'publish',
    'comment_status' => 'open',
    'post_author'    => get_current_user_id(),
    'post_parent'    => $topic_id,
)
```

Khi tạo mới, Tutor bổ sung `menu_order`, rồi gọi `wp_insert_post()`.

## 11. Post type được đăng ký ở đâu?

Các post type Tutor được đăng ký trong:

```text
wp-content/plugins/tutor/classes/Post_types.php
```

Các tên post type được cấu hình trong:

```text
wp-content/plugins/tutor/classes/Config.php
```

Giá trị đã dùng:

```text
Course     = courses
Topic      = topics
Lesson     = lesson
Enrollment = tutor_enrolled
```

Tutor đăng ký các post type trên hook WordPress:

```php
init
```

## 12. Hook liên quan

### Hook Tutor khi tạo/cập nhật Lesson

```php
tutor/lesson/created
tutor/lesson_update/before
tutor/lesson_update/after
```

### Hook WordPress tiêu chuẩn

Vì Topic và Lesson được tạo bằng `wp_insert_post()`, các hook WordPress tiêu chuẩn cũng tham gia, gồm:

```php
save_post_topics
save_post_lesson
save_post
wp_insert_post
wp_after_insert_post
```

Tutor còn đăng ký xử lý Lesson metadata tại:

```php
save_post_lesson
```

Không sửa source Tutor hoặc WordPress core. Nếu cần tùy biến, lắng nghe hook từ plugin riêng.

## 13. Tự kiểm tra bằng SQL

### Liệt kê Course, Topic và Lesson

```sql
SELECT
    ID,
    post_title,
    post_status,
    post_type,
    post_parent,
    menu_order
FROM wp_posts
WHERE post_type IN ('courses', 'topics', 'lesson')
ORDER BY ID;
```

### Tìm Topic của Course ID 16

```sql
SELECT ID, post_title, post_type, post_parent, menu_order
FROM wp_posts
WHERE post_type = 'topics'
  AND post_parent = 16
ORDER BY menu_order;
```

### Tìm Lesson của Topic ID 19

```sql
SELECT ID, post_title, post_type, post_parent, menu_order
FROM wp_posts
WHERE post_type = 'lesson'
  AND post_parent = 19
ORDER BY menu_order;
```

### JOIN một cấp cha-con

```sql
SELECT
    parent.ID AS parent_id,
    parent.post_title AS parent_title,
    parent.post_type AS parent_type,
    child.ID AS child_id,
    child.post_title AS child_title,
    child.post_type AS child_type,
    child.menu_order
FROM wp_posts AS child
INNER JOIN wp_posts AS parent
    ON parent.ID = child.post_parent
WHERE child.ID = 19
   OR child.post_parent IN (16, 19)
ORDER BY child.ID;
```

### Truy vấn toàn bộ cây Course -> Topic -> Lesson

```sql
SELECT
    course.ID AS course_id,
    course.post_title AS course_title,
    topic.ID AS topic_id,
    topic.post_title AS topic_title,
    topic.menu_order AS topic_order,
    lesson.ID AS lesson_id,
    lesson.post_title AS lesson_title,
    lesson.menu_order AS lesson_order
FROM wp_posts AS course
LEFT JOIN wp_posts AS topic
    ON topic.post_parent = course.ID
   AND topic.post_type = 'topics'
LEFT JOIN wp_posts AS lesson
    ON lesson.post_parent = topic.ID
   AND lesson.post_type = 'lesson'
WHERE course.ID = 16
  AND course.post_type = 'courses'
ORDER BY topic.menu_order, lesson.menu_order;
```

Các truy vấn trên chỉ đọc dữ liệu.

## 14. Tự kiểm tra bằng WP-CLI

```powershell
cd D:\xampp-new\htdocs\onthi-lab
```

Liệt kê Course:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" post list --post_type=courses --fields=ID,post_title,post_status,post_parent,menu_order
```

Liệt kê Topic:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" post list --post_type=topics --fields=ID,post_title,post_status,post_parent,menu_order
```

Liệt kê Lesson:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" post list --post_type=lesson --fields=ID,post_title,post_status,post_parent,menu_order
```

## 15. Điều cần ghi nhớ

1. Course, Topic và Lesson đều được lưu trong `wp_posts`.
2. Course dùng `post_type = courses`.
3. Topic dùng `post_type = topics` và `post_parent = Course ID`.
4. Lesson dùng `post_type = lesson` và `post_parent = Topic ID`.
5. `menu_order` lưu thứ tự trong Curriculum.
6. Topic và Lesson có thể là `publish` trong khi Course vẫn là `draft`.
7. Nút Save Draft của Course không lưu khối Topic đang chỉnh dở.
8. Topic phải được xác nhận bằng nút OK.
9. Lesson phải được xác nhận bằng nút Save trong cửa sổ Lesson.
10. Tutor dùng `wp_insert_post()`, nên các hook WordPress tiêu chuẩn vẫn chạy.

## 16. Câu hỏi ôn tập

1. Muốn tìm các Topic của Course ID `16`, điều kiện SQL cần kiểm tra là gì?
2. Tại sao Lesson ID `20` có `post_parent = 19` thay vì `16`?
3. Muốn tìm Course chứa Lesson ID `20`, cần đi qua đối tượng trung gian nào?
4. `menu_order = 1` của Lesson có ý nghĩa gì?
5. Vì sao Topic biến mất sau F5 khi chỉ nhấn Save as Draft của Course?
6. Action AJAX nào tạo Topic và action nào tạo Lesson?
7. Hook Tutor nào chạy sau khi một Lesson mới được tạo?
8. Vì sao không thể chỉ dựa vào `post_status = publish` của Lesson để kết luận học viên có quyền truy cập?
