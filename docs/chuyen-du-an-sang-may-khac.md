# Chuyển `onthi-lab` sang máy khác

Tài liệu này mô tả snapshot ngày **06-09-2026**, tại điểm đang học tích hợp Tutor LMS với WooCommerce.

## 1. Những gì được lưu trong Git

- WordPress 7.1.
- Tutor LMS 4.0.7, bao gồm toàn bộ source plugin.
- WooCommerce 11.1.0, bao gồm toàn bộ source plugin.
- Các theme đang có trong project.
- File `.htaccess` của site `onthi-lab`.
- Tài liệu học trong `docs`.

Việc commit source plugin giúp máy mới có đúng phiên bản đang học. Trạng thái plugin active, Course, Product, enrollment và các thiết lập vẫn nằm trong database, vì vậy chỉ clone Git là chưa đủ.

## 2. Những gì không được lưu trong Git

- `wp-config.php`: chứa thông tin kết nối database và security salts.
- Database dump: chứa email, password hash, session và dữ liệu tài khoản.
- `wp-content/uploads`: chứa file do WordPress và plugin sinh ra.
- Cache, log và thư mục cập nhật tạm thời.
- Các file trong `local-backups`.

Các mục này được `.gitignore` loại khỏi commit. Không đẩy database dump hoặc gói uploads lên repository công khai.

## 3. Bộ backup cần chép riêng

Hai file mới đã được tạo trong:

```text
D:\xampp-new\htdocs\onthi-lab\local-backups\onthi_lab-2026-09-06.sql
D:\xampp-new\htdocs\onthi-lab\local-backups\onthi-lab-uploads-2026-09-06.zip
```

Thông tin đối chiếu:

| File | Kích thước | SHA-256 |
| --- | ---: | --- |
| `onthi_lab-2026-09-06.sql` | 745091 byte | `95628B226205CFAD594E9C1AAF1F736489E3EADE19F235E92A3692E5510A40D7` |
| `onthi-lab-uploads-2026-09-06.zip` | 9568 byte | `C519D1769753EC9644A0E3680A6FD8C6ED7548AC638EF666BC516EE8DC2AA578` |

Phải chép hai file này bằng USB hoặc nơi lưu trữ cá nhân an toàn trước khi rời máy cũ. Git không chứa và không vận chuyển chúng.

Có thể kiểm tra checksum trên máy mới bằng PowerShell:

```powershell
Get-FileHash -Algorithm SHA256 .\onthi_lab-2026-09-06.sql
Get-FileHash -Algorithm SHA256 .\onthi-lab-uploads-2026-09-06.zip
```

## 4. Snapshot tại điểm dừng

### Plugin và cấu hình chính

| Thành phần | Trạng thái |
| --- | --- |
| WordPress | `7.1` |
| Tutor LMS | `4.0.7`, active |
| WooCommerce | `11.1.0`, active |
| Tutor eCommerce Engine | WooCommerce, giá trị nội bộ `wc` |
| WooCommerce currency | `VND` |
| WooCommerce default country | `VN` |
| WooCommerce Order | Chưa có Order |

### Dữ liệu thực hành quan trọng

| ID | Loại | Dữ liệu |
| ---: | --- | --- |
| `16` | Course | `Khóa học WordPress thực hành`, `publish`, loại giá `paid` |
| `19` | Topic | `Chủ đề 1 - Nền tảng WordPress`, cha là Course `16` |
| `20` | Lesson | `Bài 1 - WordPress lưu dữ liệu ở đâu?`, cha là Topic `19` |
| `21` | Enrollment | User `2`, Course `16`, trạng thái `completed` |
| `23` | Page | `Shop`, slug `shop` |
| `24` | Page | `Cart`, slug `cart-2` |
| `25` | Page | `Checkout`, slug `checkout-2` |
| `26` | Page | `My account`, slug `my-account` |
| `27` | Page | `Refund and Returns Policy`, trạng thái `draft` |
| `28` | Product | `Khóa học WordPress thực hành`, `publish`, giá `500000` VND |

Course và Product đang liên kết bằng:

```text
Course ID 16
├── _tutor_course_price_type = paid
└── _tutor_course_product_id = 28
```

Product ID `28` là Simple Product, `virtual = yes`, `sold_individually = yes`, giá hiện tại `500000`.

### Điểm thực hành đã xác nhận

- Khách chưa đăng nhập mở Course thấy giá `500.000 ₫` và nút **Add to cart**.
- User `hocvien_bai2` có enrollment `completed` và thấy nút **Start learning**.
- Chưa bấm **Start learning** để xác nhận Lesson sau lần đổi sang WooCommerce.
- Chưa thêm Product vào Cart, chưa Checkout và chưa tạo Order.
- Chưa cấu hình hay thử payment gateway.

Điểm tiếp tục được ghi chi tiết trong `docs/bai-hoc/bai-07-woocommerce-va-lien-ket-course-product.md`.

## 5. Khôi phục trên máy mới

### Bước 1: chuẩn bị source

Clone repository vào cùng đường dẫn để không phải đổi URL:

```text
D:\xampp-new\htdocs\onthi-lab
```

WordPress, Tutor LMS và WooCommerce đã nằm trong Git. Không chạy trình cài WordPress và không cài lại plugin bằng wp-admin.

### Bước 2: tạo database trống

Trong phpMyAdmin, tạo database:

```text
onthi_lab
```

Trước khi import, phải bảo đảm đây là database mới hoặc trống. Không chọn database tham khảo `onthi`.

### Bước 3: import database

Chọn database `onthi_lab`, mở tab **Import**, rồi chọn:

```text
onthi_lab-2026-09-06.sql
```

Bản dump không chứa `DROP TABLE` hoặc `DROP DATABASE`. Nếu database đã có bảng trùng tên, import có thể báo lỗi; hãy dùng một database trống.

### Bước 4: khôi phục uploads

Giải nén `onthi-lab-uploads-2026-09-06.zip`. Gói zip chứa thư mục gốc `uploads`; đặt thư mục đó vào:

```text
D:\xampp-new\htdocs\onthi-lab\wp-content\uploads
```

Kết quả phải có đường dẫn:

```text
D:\xampp-new\htdocs\onthi-lab\wp-content\uploads\woocommerce-placeholder.webp
```

Không tạo thành đường dẫn lồng `wp-content\uploads\uploads`.

### Bước 5: tạo `wp-config.php`

Sao chép `wp-config-sample.php` thành `wp-config.php`, sau đó cấu hình tối thiểu:

```php
define( 'DB_NAME', 'onthi_lab' );
define( 'DB_USER', 'root' );
define( 'DB_PASSWORD', '' );
define( 'DB_HOST', 'localhost' );
```

Thay security keys/salts mẫu bằng giá trị riêng. Không commit `wp-config.php`.

### Bước 6: xử lý URL nếu đường dẫn local thay đổi

Nếu máy mới vẫn dùng `http://localhost/onthi-lab`, bỏ qua bước này.

Nếu URL thay đổi, sau khi import hãy dùng WP-CLI để thay URL theo cách hiểu serialized data. Ví dụ URL mới là `http://localhost/onthi-lab-moi`:

```powershell
cd D:\xampp-new\htdocs\onthi-lab-moi

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  search-replace `
  "http://localhost/onthi-lab" `
  "http://localhost/onthi-lab-moi" `
  --all-tables-with-prefix `
  --skip-columns=guid `
  --dry-run
```

Chỉ bỏ `--dry-run` và chạy lại sau khi đã xem kết quả thử, xác nhận đúng database và đúng hai URL.

## 6. Kiểm tra sau khi khôi phục

Mở PowerShell tại source và chạy:

```powershell
cd D:\xampp-new\htdocs\onthi-lab

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  config get DB_NAME

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  plugin list --fields=name,status,version
```

Kết quả phải cho thấy:

- Database là `onthi_lab`.
- Tutor LMS `4.0.7` active.
- WooCommerce `11.1.0` active.

Kiểm tra cấu hình Tutor và liên kết Course bằng:

```powershell
& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  eval "echo 'monetize_by=' . tutor_utils()->get_option('monetize_by','free') . PHP_EOL; echo 'price_type=' . get_post_meta(16,'_tutor_course_price_type',true) . PHP_EOL; echo 'product_id=' . get_post_meta(16,'_tutor_course_product_id',true) . PHP_EOL;"
```

Kết quả mong đợi:

```text
monetize_by=wc
price_type=paid
product_id=28
```

Kiểm tra trên giao diện:

1. Mở `http://localhost/onthi-lab`.
2. Mở Course `http://localhost/onthi-lab/courses/khoa-hoc-wordpress-thuc-hanh/` khi chưa đăng nhập; phải thấy giá và **Add to cart**.
3. Đăng nhập bằng `hocvien_bai2`; Course phải hiện **Start learning**.
4. Chưa thử mua hàng cho đến khi các kiểm tra trên đều đúng.

## 7. Bước học tiếp theo

Sau khi khôi phục và xác minh:

1. Đăng nhập bằng `hocvien_bai2`.
2. Nhấn **Start learning**.
3. Xác nhận Lesson ID `20` mở được và không yêu cầu mua lại Course.
4. Sau đó mới bắt đầu luồng mua hàng bằng một tài khoản mới: Add to cart → Cart → Checkout → Order → Enrollment.

Không dùng user `hocvien_bai2` để thử mua vì user này đã có enrollment `completed`.
