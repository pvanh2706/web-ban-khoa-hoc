# Bài 2: WordPress lưu User, role và user metadata như thế nào?

## Mục tiêu

Sau bài này, bạn có thể:

- Phân biệt dữ liệu trong `wp_users` và `wp_usermeta`.
- Hiểu quan hệ giữa user, metadata và tác giả của Page.
- Đọc dữ liệu PHP serialized dùng để lưu role.
- Phân biệt role, capability và user level.
- Biết các hàm, class và hook chính khi WordPress tạo user và gán role.
- Tự kiểm tra dữ liệu bằng câu lệnh SQL chỉ đọc.

## 1. Hai bảng lưu dữ liệu user

### `wp_users`

`wp_users` chứa dữ liệu nhận dạng và đăng nhập chính:

| Cột | Ý nghĩa |
| --- | --- |
| `ID` | Định danh duy nhất của user |
| `user_login` | Tên đăng nhập |
| `user_pass` | Mật khẩu đã băm |
| `user_nicename` | Tên thân thiện dùng trong URL |
| `user_email` | Email |
| `user_registered` | Thời gian đăng ký |
| `display_name` | Tên hiển thị |

Không truy vấn, sao chép hoặc sửa trực tiếp `user_pass`. WordPress quản lý việc tạo và kiểm tra password hash thông qua API của core.

### `wp_usermeta`

`wp_usermeta` chứa dữ liệu mở rộng:

| Cột | Ý nghĩa |
| --- | --- |
| `umeta_id` | Khóa chính của dòng metadata |
| `user_id` | ID của user trong `wp_users` |
| `meta_key` | Tên metadata |
| `meta_value` | Giá trị metadata |

Quan hệ giữa hai bảng:

```text
wp_users.ID = wp_usermeta.user_id
```

Đây là quan hệ một-nhiều: một user có thể có nhiều dòng metadata.

## 2. Kết quả thực hành

User giả lập đã tạo:

| Thuộc tính | Giá trị |
| --- | --- |
| `ID` | `2` |
| `user_login` | `hocvien_bai2` |
| `user_nicename` | `hocvien_bai2` |
| `display_name` | `Bài 2 Học viên` |
| Role | `subscriber` |
| User level | `0` |

Số dòng thay đổi:

- `wp_users`: từ 1 lên 2 dòng.
- `wp_usermeta`: từ 20 lên 35 dòng.

WordPress tạo 15 dòng metadata cho user mới, gồm các key như:

```text
nickname
first_name
last_name
description
rich_editing
syntax_highlighting
admin_color
show_admin_bar_front
locale
wp_capabilities
wp_user_level
```

Không đọc hoặc chia sẻ giá trị của `session_tokens` và `wp_application_passwords`.

## 3. Role được lưu dưới dạng PHP serialized data

Giá trị role thực tế:

```text
a:1:{s:10:"subscriber";b:1;}
```

Cách đọc:

```text
a:1        = array có một phần tử
s:10       = string dài 10 ký tự
subscriber = tên role
b:1        = boolean true
```

Ý nghĩa sau khi unserialize tương đương:

```php
[
    'subscriber' => true,
]
```

Key `wp_capabilities` có tiền tố `wp_` vì table prefix của website là `wp_`. Với website dùng prefix khác, tên key quyền có thể thay đổi theo prefix đó.

## 4. Role, capability và user level

- **Role** là nhóm quyền có tên, ví dụ `administrator`, `editor`, `author`, `subscriber`.
- **Capability** là một quyền cụ thể, ví dụ đọc nội dung, sửa Page hoặc quản lý plugin.
- **User level** là cơ chế cũ được giữ lại vì tương thích.

Code hiện đại nên kiểm tra quyền bằng capability:

```php
current_user_can( 'edit_pages' )
```

Không nên chỉ kiểm tra tên role hoặc giá trị `wp_user_level` để quyết định user được làm gì.

Subscriber mặc định có quyền đọc nhưng không có các capability quản trị như `edit_pages`, `install_plugins` hoặc `list_users`.

## 5. Quan hệ giữa Page và tác giả

Page ID `7` trong Bài 1 có:

```text
wp_posts.post_author = 1
```

Giá trị này liên kết trực tiếp với:

```text
wp_users.ID = 1
```

Sơ đồ tổng hợp:

```text
wp_posts.post_author
        |
        v
wp_users.ID
        |
        v
wp_usermeta.user_id
```

`wp_usermeta` lưu thuộc tính mở rộng và quyền; nó không phải bảng dùng trực tiếp làm khóa tác giả trong `wp_posts`.

## 6. Tự kiểm tra bằng SQL

Kết nối MariaDB:

```powershell
& "D:\xampp-new\mysql\bin\mysql.exe" --default-character-set=utf8mb4 -u root onthi_lab
```

### Xem user mà không đọc mật khẩu

```sql
SELECT
    ID,
    user_login,
    user_nicename,
    display_name,
    user_registered,
    user_status
FROM wp_users
WHERE ID = 2;
```

### Xem metadata không nhạy cảm

```sql
SELECT umeta_id, user_id, meta_key, meta_value
FROM wp_usermeta
WHERE user_id = 2
  AND meta_key NOT IN ('session_tokens', 'wp_application_passwords')
ORDER BY umeta_id;
```

### Xem riêng role và user level

```sql
SELECT user_id, meta_key, meta_value
FROM wp_usermeta
WHERE user_id = 2
  AND meta_key IN ('wp_capabilities', 'wp_user_level');
```

### Liên kết Page với tác giả

```sql
SELECT
    p.ID AS page_id,
    p.post_title,
    p.post_author,
    u.user_login,
    u.display_name
FROM wp_posts AS p
LEFT JOIN wp_users AS u
    ON u.ID = p.post_author
WHERE p.ID = 7;
```

Các truy vấn trên chỉ đọc dữ liệu. Thoát bằng:

```sql
exit;
```

## 7. Luồng xử lý khi tạo user

```text
Users -> Add New
    -> edit_user()
    -> wp_insert_user()
    -> ghi dữ liệu chính vào wp_users
    -> ghi metadata vào wp_usermeta
    -> WP_User::set_role()
    -> gán role subscriber
```

## 8. File, class và hàm quan trọng

| Thành phần | File | Vai trò |
| --- | --- | --- |
| `edit_user()` | `wp-admin/includes/user.php` | Nhận và kiểm tra dữ liệu từ form quản trị |
| `wp_insert_user()` | `wp-includes/user.php` | Tạo user và metadata |
| `wp_create_user()` | `wp-includes/user.php` | Hàm đơn giản hóa việc tạo user |
| `wp_update_user()` | `wp-includes/user.php` | Cập nhật user hiện có |
| `WP_User::set_role()` | `wp-includes/class-wp-user.php` | Gán role cho user |
| `add_user_meta()` | `wp-includes/user.php` | Thêm user metadata |
| `get_user_meta()` | `wp-includes/user.php` | Đọc user metadata |
| `update_user_meta()` | `wp-includes/user.php` | Cập nhật user metadata |

Chỉ đọc các file core để học; không chỉnh sửa chúng.

## 9. Hook liên quan

Trước khi dữ liệu user được ghi:

```php
wp_pre_insert_user_data
insert_user_meta
```

Sau khi tạo user:

```php
user_register
```

Khi gán hoặc thay đổi role:

```php
add_user_role
remove_user_role
set_user_role
```

Khi cập nhật user đã tồn tại:

```php
profile_update
```

Plugin riêng có thể lắng nghe các hook này. Không sửa `wp-includes/user.php` hoặc `wp-includes/class-wp-user.php`.

## 10. Kiểm tra quyền bằng giao diện

Phần này có thể thực hiện khi ôn lại:

1. Giữ cửa sổ đang đăng nhập bằng admin.
2. Mở một cửa sổ ẩn danh.
3. Đăng nhập bằng `hocvien_bai2`.
4. Quan sát xem có các menu Trang, Plugin hoặc Thành viên hay không.
5. Thử mở `http://localhost/onthi-lab/wp-admin/post-new.php?post_type=page`.
6. Xác nhận WordPress từ chối thao tác nếu user không có capability phù hợp.

Đăng nhập sẽ tạo hoặc cập nhật `session_tokens` trong `wp_usermeta`. Đây là dữ liệu xác thực nhạy cảm; chỉ cần biết nơi lưu, không xem giá trị.

## 11. Điều cần ghi nhớ

1. Thông tin user chính nằm trong `wp_users`.
2. Thông tin mở rộng và role nằm trong `wp_usermeta`.
3. `wp_users.ID` liên kết với `wp_usermeta.user_id`.
4. `wp_posts.post_author` liên kết với `wp_users.ID`.
5. Role được lưu dưới dạng serialized data trong `{prefix}_capabilities`.
6. Quyền thực tế được quyết định bởi capability.
7. Không dùng `wp_user_level` làm cơ chế phân quyền mới.
8. Không đọc hoặc chia sẻ password hash, session token và application password.
9. Không sửa trực tiếp WordPress core.

## 12. Câu hỏi ôn tập

1. `wp_users` và `wp_usermeta` khác nhau như thế nào?
2. Page liên kết với tác giả qua hai cột nào?
3. Chuỗi `a:1:{s:10:"subscriber";b:1;}` biểu thị điều gì?
4. Vì sao nên kiểm tra capability thay vì chỉ kiểm tra tên role?
5. User level có nên được dùng làm hệ thống phân quyền mới không?
6. Khi đăng nhập, loại metadata nhạy cảm nào có thể được tạo?
7. Hook nào phù hợp nếu plugin cần phản ứng sau khi một user mới được tạo?
