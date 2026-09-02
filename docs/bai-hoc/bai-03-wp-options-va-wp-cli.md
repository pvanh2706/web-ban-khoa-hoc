# Bài 3: `wp_options` và WP-CLI

## Mục tiêu

Sau bài này, bạn có thể:

- Phân biệt option toàn website với post metadata và user metadata.
- Hiểu cấu trúc cơ bản của bảng `wp_options`.
- Biết ý nghĩa của cột `autoload`.
- Đọc và cập nhật một option an toàn qua wp-admin hoặc WP-CLI.
- Biết các hàm và hook chính trong Options API.
- Nhận biết những option nhạy cảm không nên thay đổi tùy tiện.

## 1. `wp_options` dùng để làm gì?

`wp_options` lưu cấu hình áp dụng cho toàn website.

So sánh:

| Vị trí | Phạm vi dữ liệu |
| --- | --- |
| `wp_postmeta` | Gắn với một post, Page hoặc custom post type |
| `wp_usermeta` | Gắn với một user |
| `wp_options` | Áp dụng cho toàn website |

Cấu trúc chính:

| Cột | Ý nghĩa |
| --- | --- |
| `option_id` | ID của dòng option |
| `option_name` | Tên option |
| `option_value` | Giá trị option |
| `autoload` | Chế độ tự động nạp option |

`option_name` được thiết kế để định danh duy nhất một option. Khi gọi `update_option()` cho một option hiện có, WordPress cập nhật dòng đó thay vì tạo dòng trùng tên.

## 2. Một số option hệ thống

Trạng thái đã quan sát trước thực hành:

| Option | Giá trị |
| --- | --- |
| `blogname` | `web bán hàng` |
| `blogdescription` | Trống |
| `siteurl` | `http://localhost/onthi-lab` |
| `home` | `http://localhost/onthi-lab` |
| `posts_per_page` | `10` |
| `show_on_front` | `posts` |
| `page_on_front` | `0` |
| `page_for_posts` | `0` |
| `start_of_week` | `1` |
| `permalink_structure` | `/index.php/%year%/%monthnum%/%day%/%postname%/` |

Không thay đổi `siteurl` hoặc `home` khi chưa hiểu rõ tác động. Giá trị sai có thể khiến website chuyển hướng sai hoặc không vào được wp-admin.

`wp_options` còn có thể chứa:

- Cấu hình theme và plugin.
- Role definitions.
- Scheduled task data.
- Cache và transient.
- Token hoặc khóa cấu hình của plugin.

Vì vậy, không nên xuất toàn bộ `wp_options` ra nơi công khai.

## 3. Kết quả thực hành bằng wp-admin

Trong **Cài đặt → Tổng quan**, `blogdescription` đã được đổi thành:

```text
Website thực hành WordPress
```

Kết quả database:

| Thuộc tính | Giá trị |
| --- | --- |
| `option_id` | `5` |
| `option_name` | `blogdescription` |
| `option_value` | `Website thực hành WordPress` |
| `autoload` | `on` |

Database vẫn chỉ có một dòng mang tên `blogdescription`. Trang chủ phản hồi HTTP `200` và hiển thị giá trị mới.

Điều này cho thấy việc cập nhật option không tạo thêm một dòng trùng lặp.

## 4. `autoload` là gì?

Option được autoload có thể được WordPress nạp sẵn khi khởi động request để tránh nhiều truy vấn nhỏ lặp lại.

Trạng thái quan sát được:

| `autoload` | Số option | Tổng dung lượng giá trị |
| --- | ---: | ---: |
| `on` | 101 | 48.49 KiB |
| `auto` | 20 | 1.53 KiB |
| `off` | 39 | 1579.15 KiB |

Không nên đặt mọi option thành autoload. Nhiều dữ liệu lớn được autoload có thể làm tăng bộ nhớ và thời gian xử lý mỗi request.

Không sửa cột `autoload` trực tiếp bằng SQL trong bài này.

## 5. Tự kiểm tra bằng SQL

Kết nối MariaDB:

```powershell
& "D:\xampp-new\mysql\bin\mysql.exe" --default-character-set=utf8mb4 -u root onthi_lab
```

### Đọc các option cơ bản

```sql
SELECT option_id, option_name, option_value, autoload
FROM wp_options
WHERE option_name IN (
    'blogname',
    'blogdescription',
    'siteurl',
    'home'
)
ORDER BY option_id;
```

### Kiểm tra một option có bị trùng không

```sql
SELECT COUNT(*) AS option_rows
FROM wp_options
WHERE option_name = 'blogdescription';
```

Kết quả mong đợi là `1`.

### Thống kê autoload

```sql
SELECT
    autoload,
    COUNT(*) AS option_count,
    ROUND(SUM(LENGTH(option_value)) / 1024, 2) AS value_kib
FROM wp_options
GROUP BY autoload
ORDER BY autoload;
```

Các truy vấn trên chỉ đọc dữ liệu.

## 6. Đọc option bằng WP-CLI

Chuyển đến đúng project:

```powershell
cd D:\xampp-new\htdocs\onthi-lab
```

Đọc tiêu đề, khẩu hiệu và URL:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" option get blogname
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" option get blogdescription
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" option get home
```

`option get` chỉ đọc dữ liệu.

## 7. Cập nhật option bằng WP-CLI

Phần thực hành này được giữ lại để tự làm sau:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" option update blogdescription "Website thực hành WordPress bằng WP-CLI"
```

Đọc lại để xác nhận:

```powershell
& "D:\xampp-new\php\php.exe" "D:\xampp-new\wp-cli\wp-cli.phar" option get blogdescription
```

Lệnh `option update` là thao tác ghi. Chỉ dùng với `blogdescription` trong bài tập này. Không thử với `siteurl` hoặc `home`.

Sau khi cập nhật, chạy lại truy vấn SQL để kiểm tra:

- `option_id` vẫn là `5`.
- Chỉ có một dòng `blogdescription`.
- `option_value` đổi thành giá trị mới.

## 8. Luồng xử lý khi lưu Settings

```text
Settings -> General
    -> POST wp-admin/options.php
    -> kiểm tra quyền và nonce
    -> kiểm tra option được phép cập nhật
    -> update_option()
    -> cập nhật wp_options
    -> cập nhật option cache
    -> chạy hook sau cập nhật
```

## 9. File và hàm quan trọng

| Thành phần | File | Vai trò |
| --- | --- | --- |
| Form General Settings | `wp-admin/options-general.php` | Hiển thị form cấu hình chung |
| Trình xử lý Settings | `wp-admin/options.php` | Kiểm tra và lưu các option được phép |
| `get_option()` | `wp-includes/option.php` | Đọc option |
| `add_option()` | `wp-includes/option.php` | Thêm option mới |
| `update_option()` | `wp-includes/option.php` | Thêm hoặc cập nhật option |
| `delete_option()` | `wp-includes/option.php` | Xóa option |

Không sửa trực tiếp các file này.

## 10. Hook liên quan

Với option `blogdescription`, các hook đáng chú ý là:

```php
pre_update_option_blogdescription
pre_update_option
update_option_blogdescription
updated_option
```

Thứ tự khái quát:

```text
giá trị mới
    -> pre_update_option_blogdescription
    -> pre_update_option
    -> ghi database và cập nhật cache
    -> update_option_blogdescription
    -> updated_option
```

Nếu giá trị mới giống giá trị cũ, `update_option()` thường trả về `false` và không cần ghi database.

## 11. Điều cần ghi nhớ

1. `wp_options` lưu cấu hình toàn website.
2. Option không thuộc riêng một Page hoặc user.
3. Dùng Options API thay vì viết SQL trực tiếp trong code ứng dụng.
4. `get_option()` dùng để đọc; `update_option()` dùng để thêm hoặc cập nhật.
5. Không thay đổi `home` và `siteurl` tùy tiện.
6. Không để dữ liệu lớn autoload nếu không cần thiết.
7. Không công khai toàn bộ bảng `wp_options` vì plugin có thể lưu dữ liệu nhạy cảm trong đó.
8. WP-CLI phải được chạy trong đúng thư mục project.

## 12. Câu hỏi ôn tập

1. `wp_options` khác `wp_postmeta` và `wp_usermeta` ở điểm nào?
2. Vì sao cập nhật `blogdescription` không tạo dòng mới?
3. `autoload` ảnh hưởng như thế nào đến mỗi request WordPress?
4. Vì sao không nên autoload một option có giá trị rất lớn?
5. Hàm nào dùng để đọc option và hàm nào dùng để cập nhật?
6. Vì sao thay đổi sai `home` hoặc `siteurl` có thể làm website khó truy cập?
7. Hook động nào chạy trước khi `blogdescription` được cập nhật?
