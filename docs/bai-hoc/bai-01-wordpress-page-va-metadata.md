# Bài 1: WordPress lưu Page và metadata trong database như thế nào?

## Mục tiêu

Sau bài này, bạn có thể:

- Giải thích vì sao một Page được lưu trong `wp_posts`.
- Phân biệt dữ liệu chính của Page với metadata trong `wp_postmeta`.
- Hiểu quan hệ giữa `wp_posts.ID` và `wp_postmeta.post_id`.
- Nhận biết revision và ý nghĩa của `post_parent`.
- Biết các hàm, class và hook chính tham gia quá trình lưu Page.
- Tự dùng câu lệnh SQL chỉ đọc để kiểm tra kết quả.

## Môi trường thực hành

- Website: `http://localhost/onthi-lab`
- Source: `D:\xampp-new\htdocs\onthi-lab`
- Database: `onthi_lab`
- Table prefix: `wp_`
- WordPress: `7.1`
- PHP CLI: `8.2.12`
- WP-CLI: `2.12.0`

Không chỉnh sửa trực tiếp WordPress core. Code tự viết trong các bài sau phải nằm trong plugin riêng hoặc child theme khi phù hợp.

## 1. Page là một post type

WordPress dùng bảng `wp_posts` để lưu nhiều loại đối tượng, không chỉ bài viết blog.

Một số giá trị thường gặp của cột `post_type`:

| `post_type` | Ý nghĩa |
| --- | --- |
| `post` | Bài viết |
| `page` | Trang |
| `revision` | Phiên bản lưu lại của bài viết hoặc Page |
| `attachment` | Tệp trong Media Library |
| `nav_menu_item` | Mục menu |
| `wp_global_styles` | Cấu hình Global Styles của block theme |

Vì vậy, một Page được nhận diện chủ yếu bởi:

```text
wp_posts.post_type = 'page'
```

## 2. Dữ liệu chính của Page trong `wp_posts`

Các cột quan trọng:

| Cột | Ý nghĩa |
| --- | --- |
| `ID` | Khóa chính, định danh duy nhất của Page |
| `post_author` | ID của user tạo hoặc sửa Page |
| `post_date` | Thời gian tạo |
| `post_modified` | Thời gian cập nhật gần nhất |
| `post_title` | Tiêu đề |
| `post_content` | Nội dung |
| `post_excerpt` | Tóm tắt, nếu có |
| `post_name` | Slug dùng trong URL |
| `post_status` | Trạng thái như `draft`, `publish`, `trash` |
| `post_type` | Loại đối tượng; Page có giá trị `page` |
| `post_parent` | ID đối tượng cha, tùy loại dữ liệu |
| `menu_order` | Thứ tự, thường dùng với Page |

### Nội dung Gutenberg

Gutenberg không chỉ lưu văn bản thuần. `post_content` chứa HTML kèm các comment đánh dấu block, ví dụ:

```html
<!-- wp:paragraph -->
<p>Nội dung của Page</p>
<!-- /wp:paragraph -->
```

Các dấu `<!-- wp:... -->` giúp block editor dựng lại đúng loại block và thuộc tính khi mở Page để chỉnh sửa.

## 3. Metadata trong `wp_postmeta`

Metadata là dữ liệu bổ sung gắn với một post, Page hoặc custom post type.

Các cột chính:

| Cột | Ý nghĩa |
| --- | --- |
| `meta_id` | Khóa chính của từng dòng metadata |
| `post_id` | ID của đối tượng trong `wp_posts` |
| `meta_key` | Tên metadata |
| `meta_value` | Giá trị metadata |

Quan hệ giữa hai bảng:

```text
wp_posts.ID = wp_postmeta.post_id
```

Đây là quan hệ một-nhiều: một Page có thể có nhiều dòng metadata.

```text
wp_posts.ID = 7
       |
       +-- wp_postmeta.post_id = 7, meta_key = _edit_lock
       +-- wp_postmeta.post_id = 7, meta_key = lab_lesson
       +-- wp_postmeta.post_id = 7, meta_key = _edit_last
```

### Metadata bắt đầu bằng dấu gạch dưới

Các key bắt đầu bằng `_`, chẳng hạn `_edit_lock`, thường được WordPress xem là metadata nội bộ/protected và không hiện như trường tùy biến thông thường.

Key `lab_lesson` không bắt đầu bằng `_`, vì vậy có thể quản lý từ giao diện Custom Fields.

## 4. Kết quả thực hành

Page đã tạo:

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `7` |
| `post_title` | `Bài 1 - Dữ liệu Page` |
| `post_name` | `bai-1-du-lieu-page` |
| `post_type` | `page` |
| `post_status` | `publish` |
| `post_parent` | `0` |

Metadata của Page ID `7`:

| `meta_key` | `meta_value` | Ý nghĩa |
| --- | --- | --- |
| `_edit_lock` | Dấu thời gian và user ID | Khóa biên tập tạm thời |
| `lab_lesson` | `wordpress-page-meta` | Metadata được tạo trong bài học |
| `_edit_last` | `1` | User ID sửa Page gần nhất |

Revision được tạo:

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `9` |
| `post_type` | `revision` |
| `post_status` | `inherit` |
| `post_parent` | `7` |

`post_parent = 7` nghĩa là revision ID `9` thuộc Page ID `7`.

Trong lần thực hành này, tổng số dòng trong `wp_posts` tăng từ 6 lên 9:

- ID `7`: Page thực hành.
- ID `8`: `wp_global_styles` của theme Twenty Twenty-Five.
- ID `9`: revision của Page.

Điều này chứng minh một thao tác trong giao diện có thể tạo nhiều hơn một dòng database.

## 5. Tự kiểm tra bằng SQL

Mở PowerShell và kết nối MariaDB:

```powershell
& "D:\xampp-new\mysql\bin\mysql.exe" --default-character-set=utf8mb4 -u root onthi_lab
```

### Xem Page

```sql
SELECT
    ID,
    post_author,
    post_date,
    post_modified,
    post_title,
    post_name,
    post_status,
    post_type,
    post_parent
FROM wp_posts
WHERE ID = 7;
```

### Xem nội dung Gutenberg

```sql
SELECT post_content
FROM wp_posts
WHERE ID = 7;
```

### Xem metadata

```sql
SELECT meta_id, post_id, meta_key, meta_value
FROM wp_postmeta
WHERE post_id = 7
ORDER BY meta_id;
```

### Xem revision của Page

```sql
SELECT ID, post_title, post_status, post_type, post_parent
FROM wp_posts
WHERE post_type = 'revision'
  AND post_parent = 7;
```

### JOIN Page với metadata

```sql
SELECT
    p.ID,
    p.post_title,
    pm.meta_id,
    pm.meta_key,
    pm.meta_value
FROM wp_posts AS p
LEFT JOIN wp_postmeta AS pm
    ON pm.post_id = p.ID
WHERE p.ID = 7
ORDER BY pm.meta_id;
```

Các câu lệnh trên chỉ đọc dữ liệu. Dùng lệnh sau để thoát:

```sql
exit;
```

## 6. Luồng xử lý khi lưu Page

Với block editor, luồng khái quát là:

```text
Block editor
    -> WordPress REST API
    -> WP_REST_Posts_Controller
    -> wp_insert_post() hoặc wp_update_post()
    -> ghi dữ liệu chính vào wp_posts
    -> API metadata ghi dữ liệu vào wp_postmeta
    -> WordPress chạy các action liên quan
```

### File, class và hàm quan trọng

| Thành phần | Vị trí | Vai trò |
| --- | --- | --- |
| `WP_REST_Posts_Controller::create_item()` | `wp-includes/rest-api/endpoints/class-wp-rest-posts-controller.php` | Nhận REST request tạo post/Page |
| `WP_REST_Posts_Controller::update_item()` | Cùng file trên | Nhận REST request cập nhật |
| `wp_insert_post()` | `wp-includes/post.php` | Tạo dữ liệu post/Page |
| `wp_update_post()` | `wp-includes/post.php` | Cập nhật dữ liệu post/Page |
| `update_post_meta()` | `wp-includes/post.php` | API cập nhật post metadata |
| `update_metadata()` | `wp-includes/meta.php` | API metadata dùng chung |
| `post_custom_meta_box()` | `wp-admin/includes/meta-boxes.php` | Dựng giao diện Custom Fields |

Chỉ đọc các file core để tìm hiểu; không sửa chúng.

## 7. Hook liên quan

### Hook khi lưu Page

Các action quan trọng:

```php
save_post_page
save_post
wp_insert_post
wp_after_insert_post
```

Trong đó:

- `save_post_page` chỉ chạy cho post type `page`.
- `save_post` chạy chung cho nhiều post type.
- `wp_after_insert_post` chạy sau khi post, taxonomy và metadata đã được lưu theo luồng tương ứng.

Filter quan trọng trước khi dữ liệu được ghi:

```php
wp_insert_post_data
```

### Hook metadata

Khi thêm hoặc cập nhật metadata, thường gặp:

```php
add_post_metadata
added_post_meta
update_post_metadata
updated_post_meta
```

Tên hook phản ánh thời điểm trước hoặc sau khi thêm/cập nhật metadata. Trong bài này chưa viết code để theo dõi hook, nên danh sách trên là các điểm mở rộng liên quan trong API, không phải log chứng minh từng hook đã chạy.

## 8. Điều cần ghi nhớ

1. Page là một dòng có `post_type = 'page'` trong `wp_posts`.
2. Nội dung Gutenberg nằm trong `post_content` dưới dạng block markup.
3. Dữ liệu mở rộng nằm trong `wp_postmeta`.
4. `wp_posts.ID` liên kết với `wp_postmeta.post_id`.
5. Một Page có thể có nhiều metadata.
6. Revision cũng nằm trong `wp_posts`, với `post_type = 'revision'`.
7. `post_parent` liên kết revision với Page gốc.
8. Một thao tác trên wp-admin có thể tạo hoặc cập nhật nhiều dòng dữ liệu.
9. Không sửa WordPress core; dùng hook trong plugin riêng hoặc child theme khi cần tùy biến.

## 9. Câu hỏi ôn tập

1. Cột nào cho biết một dòng trong `wp_posts` là Page?
2. Vì sao `lab_lesson` không cần một cột riêng trong `wp_posts`?
3. Hai cột nào dùng để liên kết `wp_posts` và `wp_postmeta`?
4. `post_parent = 7` trên revision ID `9` có nghĩa gì?
5. Vì sao tổng số dòng `wp_posts` có thể tăng nhiều hơn một khi chỉ tạo một Page?
6. Metadata bắt đầu bằng `_` thường khác metadata thông thường như thế nào?
7. Khi muốn tùy biến quá trình lưu Page, vì sao không nên sửa `wp-includes/post.php` trực tiếp?

## 10. Bài thực hành ôn lại

Không cần thực hiện ngay. Khi ôn bài, bạn có thể:

1. Mở Page ID `7` trong wp-admin.
2. Đổi `lab_lesson` từ `wordpress-page-meta` thành `wordpress-page-meta-v2`.
3. Cập nhật Page.
4. Chạy lại truy vấn `wp_postmeta` để xem giá trị mới.
5. Chạy lại truy vấn revision để kiểm tra WordPress có tạo revision mới hay không.
6. So sánh `post_modified` trước và sau khi cập nhật.

Thao tác này giúp phân biệt rõ việc cập nhật dòng Page, cập nhật metadata và tạo revision.
