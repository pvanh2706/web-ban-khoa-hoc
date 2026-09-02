# Chuyển `onthi-lab` sang máy khác

## Những gì được lưu trong Git

- WordPress 7.1.
- Tutor LMS 4.0.7, bao gồm toàn bộ source plugin.
- Các theme đang có trong project.
- File `.htaccess` của site `onthi-lab`.
- Tài liệu học trong `docs`.

Việc commit Tutor LMS giúp máy mới có đúng phiên bản plugin mà không phải tải và cài lại. Plugin vẫn cần dữ liệu trong database để biết nó đang active và để đọc Course, Lesson, enrollment cùng các thiết lập.

## Những gì không được lưu trong Git

- `wp-config.php`: chứa thông tin kết nối database và security salts.
- Database dump: chứa email, password hash, session và dữ liệu tài khoản.
- `wp-content/uploads`: nội dung người dùng tải lên.
- Cache, log và thư mục cập nhật tạm thời.

Các mục này được `.gitignore` loại khỏi commit.

## Bản sao database hiện tại

Bản sao cục bộ được tạo tại:

```text
D:\xampp-new\htdocs\onthi-lab\local-backups\onthi_lab-2026-09-02.sql
```

File này bị Git bỏ qua. Hãy chép riêng bằng USB hoặc nơi lưu trữ cá nhân an toàn; không đẩy file lên repository công khai.

## Khôi phục trên máy mới

### 1. Chuẩn bị source

Clone repository vào đúng đường dẫn:

```text
D:\xampp-new\htdocs\onthi-lab
```

Vì WordPress và Tutor LMS đã nằm trong Git, không cần chạy trình cài WordPress hoặc cài lại Tutor LMS.

### 2. Tạo database trống

Trong phpMyAdmin, tạo database:

```text
onthi_lab
```

Trước khi import, phải bảo đảm đây là database mới hoặc trống. Tuyệt đối không chọn database tham khảo `onthi`.

### 3. Import database

Chọn database `onthi_lab`, mở tab **Import**, rồi chọn file:

```text
onthi_lab-2026-09-02.sql
```

Bản dump không chứa lệnh xóa bảng. Nếu database đích đã có bảng trùng tên, quá trình import sẽ báo lỗi thay vì chủ động xóa chúng.

### 4. Tạo `wp-config.php`

Sao chép `wp-config-sample.php` thành `wp-config.php`, sau đó cấu hình tối thiểu:

```php
define( 'DB_NAME', 'onthi_lab' );
define( 'DB_USER', 'root' );
define( 'DB_PASSWORD', '' );
define( 'DB_HOST', 'localhost' );
```

Thay các security keys/salts mẫu bằng giá trị riêng. Không commit `wp-config.php`.

### 5. Kiểm tra

Mở:

```text
http://localhost/onthi-lab
```

Sau đó kiểm tra:

1. Trang chủ Lab mở được.
2. Tutor LMS đang active.
3. Course ID `16`, Topic ID `19` và Lesson ID `20` còn tồn tại.
4. Enrollment ID `21` có trạng thái `completed`.
5. `hocvien_bai2` mở được Lesson.

Có thể kiểm tra database bằng WP-CLI:

```powershell
cd D:\xampp-new\htdocs\onthi-lab

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  config get DB_NAME

& "D:\xampp-new\php\php.exe" `
  "D:\xampp-new\wp-cli\wp-cli.phar" `
  plugin list --fields=name,status,version
```

## Uploads

Tại thời điểm tạo bản chuyển máy, `wp-content/uploads` chưa có file nên không cần sao chép. Khi project có ảnh, video hoặc PDF, cần sao lưu thư mục này riêng cùng database.

## WooCommerce

WooCommerce chưa được cài hoàn chỉnh tại thời điểm tạo commit này. Thư mục giải nén tạm trong `wp-content/upgrade` không được commit. Sau khi chuyển máy và xác minh site hoạt động, tiếp tục cài WooCommerce từ `wp-admin` theo bài học.

