# Bài 7: Kích hoạt WooCommerce và liên kết Course với Product

## Mục tiêu

Sau phần thực hành này, bạn có thể:

- Phân biệt Page hệ thống của Tutor LMS và Page hệ thống của WooCommerce.
- Hiểu WooCommerce liên kết Page bằng ID lưu trong `wp_options`.
- Nhận biết ba cách Page WooCommerce được render: archive động, block và shortcode.
- Hiểu Product được lưu trong `wp_posts`, `wp_postmeta` và bảng lookup.
- Hiểu quan hệ giữa Tutor Course và WooCommerce Product.
- Kiểm tra sự khác nhau giữa khách chưa mua và học viên đã có enrollment.

## 1. Trạng thái ban đầu

Trước bài này:

- WordPress `7.1`.
- Tutor LMS `4.0.7` đang active.
- Tutor đang dùng native eCommerce, giá trị `monetize_by = tutor`.
- Course ID `16` là Course miễn phí.
- Enrollment ID `21` của user ID `2` đang ở trạng thái `completed`.
- Chưa có WooCommerce Product hoặc Order.

WooCommerce `11.1.0` đã được cài và kích hoạt qua wp-admin.

## 2. Cài plugin khác với kích hoạt plugin

- Cài plugin đưa source vào `wp-content/plugins/woocommerce`.
- Kích hoạt plugin thêm WooCommerce vào danh sách plugin active và chạy activation/setup logic.
- Trạng thái active nằm trong database, không nằm trong riêng thư mục plugin.

Sau khi kích hoạt, WooCommerce tạo hoặc chuẩn bị Page, option, role, bảng dữ liệu và scheduled action phục vụ bán hàng.

## 3. Page hệ thống được tạo

| ID | Tiêu đề | Slug | Trạng thái |
| ---: | --- | --- | --- |
| `23` | Shop | `shop` | `publish` |
| `24` | Cart | `cart-2` | `publish` |
| `25` | Checkout | `checkout-2` | `publish` |
| `26` | My account | `my-account` | `publish` |
| `27` | Refund and Returns Policy | `refund_returns` | `draft` |

Trước đó Tutor LMS đã có Page Cart ID `13`, slug `cart`, và Checkout ID `14`, slug `checkout`. WordPress yêu cầu slug Page không trùng nhau nên hai Page WooCommerce mới nhận hậu tố `-2`.

Không xóa hoặc đổi slug chỉ vì tiêu đề bị trùng. Hệ thống nhận diện Page chủ yếu bằng ID được cấu hình.

## 4. WooCommerce liên kết Page qua `wp_options`

Truy vấn:

```sql
SELECT option_name, option_value
FROM wp_options
WHERE option_name IN (
    'woocommerce_shop_page_id',
    'woocommerce_cart_page_id',
    'woocommerce_checkout_page_id',
    'woocommerce_myaccount_page_id',
    'woocommerce_terms_page_id'
)
ORDER BY option_name;
```

Kết quả:

```text
woocommerce_shop_page_id       23
woocommerce_cart_page_id       24
woocommerce_checkout_page_id   25
woocommerce_myaccount_page_id  26
woocommerce_terms_page_id      chuỗi rỗng
```

`woocommerce_terms_page_id` có dòng option nhưng `option_value` dài `0`; đó là chuỗi rỗng, không phải SQL `NULL`. Nghĩa là chưa gán Page Điều khoản.

Quan hệ tổng quát:

```text
wp_options.option_value = wp_posts.ID
```

Ví dụ `woocommerce_cart_page_id = 24` cho WooCommerce biết Page nào là Cart. Tiêu đề và slug có thể đổi mà quan hệ vẫn còn nếu Page ID không đổi.

## 5. Vì sao các Page có nội dung khác nhau?

Truy vấn quan sát:

```sql
SELECT
    ID,
    post_title,
    post_name,
    CHAR_LENGTH(post_content) AS content_length,
    LEFT(post_content, 200) AS content_preview
FROM wp_posts
WHERE ID IN (23, 24, 25, 26)
ORDER BY ID;
```

Kết quả chính:

| Page | Cách lưu/render |
| --- | --- |
| Shop ID `23` | `post_content` rỗng; WooCommerce dùng Page làm điểm neo và render product archive động. |
| Cart ID `24` | Chứa block markup bắt đầu bằng `<!-- wp:woocommerce/cart -->`. |
| Checkout ID `25` | Chứa block markup bắt đầu bằng `<!-- wp:woocommerce/checkout -->`. |
| My account ID `26` | Chứa shortcode `[woocommerce_my_account]`. |

Shop không bị sửa hay thay `post_content`. Khi request được nhận diện là Shop, WooCommerce chạy truy vấn cho post type `product`, rồi template/block của product archive dựng nội dung động.

## 6. Dữ liệu mới ngoài Page

WooCommerce thêm role:

```text
customer
shop_manager
```

WooCommerce cũng tạo nhiều bảng riêng, ví dụ:

```text
wp_woocommerce_order_items
wp_woocommerce_order_itemmeta
wp_wc_orders
wp_wc_order_addresses
wp_wc_product_meta_lookup
wp_wc_customer_lookup
wp_actionscheduler_actions
wp_actionscheduler_logs
```

Các bảng `wp_tutor_orders`, `wp_tutor_order_items` và `wp_tutor_carts` đã có từ Tutor native eCommerce. Chúng không phải bảng WooCommerce dù đều phục vụ chức năng bán hàng.

## 7. Chuyển Tutor sang WooCommerce

Trong:

```text
Tutor LMS → Settings → Monetization
```

`Select eCommerce Engine` được đổi từ **Native** sang **WooCommerce**.

Các tùy chọn sau vẫn tắt trong lần thực hành này:

- Automatically Complete WooCommerce Orders.
- Auto Redirect to Courses.
- Enable Guest Mode.

Sau khi lưu, Tutor trả về:

```text
monetize_by = wc
```

Thao tác này chọn engine xử lý giao dịch. Nó không tự tạo Product, không xóa enrollment cũ và không tự biến Course miễn phí thành Course trả phí.

## 8. Product thực hành

Product được tạo thủ công vì Tutor LMS bản miễn phí chọn Product có sẵn từ danh sách.

| Thuộc tính | Giá trị |
| --- | --- |
| ID | `28` |
| Tên | `Khóa học WordPress thực hành` |
| `post_type` | `product` |
| `post_status` | `publish` |
| Loại | Simple Product |
| Regular price | `500000` VND |
| Virtual | `yes` |
| Sold individually | `yes` |

Product là một dòng trong `wp_posts`:

```sql
SELECT ID, post_title, post_name, post_status, post_type, post_parent
FROM wp_posts
WHERE ID = 28;
```

Các thuộc tính mở rộng nằm trong `wp_postmeta`:

```sql
SELECT meta_key, meta_value
FROM wp_postmeta
WHERE post_id = 28
  AND meta_key IN (
      '_regular_price',
      '_sale_price',
      '_price',
      '_virtual',
      '_downloadable',
      '_sold_individually',
      '_stock_status'
  )
ORDER BY meta_key;
```

Kết quả quan trọng:

```text
_regular_price     500000
_price             500000
_virtual           yes
_sold_individually yes
_stock_status      instock
```

## 9. Vì sao giá xuất hiện ở nhiều nơi?

- `_regular_price`: giá niêm yết gốc.
- `_sale_price`: giá giảm, nếu có.
- `_price`: giá đang có hiệu lực để bán.
- `min_price` và `max_price`: giá trong bảng lookup để lọc và sắp xếp nhanh.

Vì Product ID `28` là Simple Product, không có sale price:

```text
_regular_price = 500000
_price         = 500000
min_price      = 500000
max_price      = 500000
```

Nếu regular price là `500000` và sale price là `400000`, `_price`, `min_price` và `max_price` sẽ phản ánh giá đang bán `400000` đối với Simple Product.

Bảng lookup có thể kiểm tra bằng:

```sql
SELECT product_id, virtual, downloadable, min_price, max_price, stock_status
FROM wp_wc_product_meta_lookup
WHERE product_id = 28;
```

Không cập nhật trực tiếp `wp_wc_product_meta_lookup`. Dùng giao diện hoặc WooCommerce CRUD API để WooCommerce đồng bộ metadata và lookup đúng cách.

## 10. Liên kết Course với Product

Course ID `16` được sửa trong Tutor Course Builder:

- Pricing chuyển từ Free sang Paid.
- WooCommerce Product được chọn là Product ID `28`.
- Course được Update thành công.

Kết quả trong `wp_postmeta`:

```text
post_id = 16 | _tutor_course_price_type = paid
post_id = 16 | _tutor_course_product_id = 28
```

Quan hệ:

```text
Course ID 16
    |
    | wp_postmeta.meta_key   = _tutor_course_product_id
    | wp_postmeta.meta_value = 28
    v
Product ID 28
```

Course và Product vẫn là hai đối tượng riêng trong `wp_posts`:

- Course dùng `post_type = courses`.
- Product dùng `post_type = product`.
- Metadata trên Course giữ ID của Product.

Truy vấn JOIN:

```sql
SELECT
    course.ID AS course_id,
    course.post_title AS course_title,
    relation.meta_value AS product_id,
    product.post_title AS product_title,
    price.meta_value AS product_price
FROM wp_posts AS course
INNER JOIN wp_postmeta AS relation
    ON relation.post_id = course.ID
   AND relation.meta_key = '_tutor_course_product_id'
INNER JOIN wp_posts AS product
    ON product.ID = CAST(relation.meta_value AS UNSIGNED)
LEFT JOIN wp_postmeta AS price
    ON price.post_id = product.ID
   AND price.meta_key = '_price'
WHERE course.ID = 16;
```

## 11. Kết quả kiểm tra quyền truy cập

Khi chưa đăng nhập, mở:

```text
http://localhost/onthi-lab/courses/khoa-hoc-wordpress-thuc-hanh/
```

Kết quả:

- Hiển thị giá `500.000 ₫`.
- Hiển thị nút **Add to cart**.

Khi đăng nhập bằng `hocvien_bai2`:

- Course hiển thị nút **Start learning**.
- Tutor không yêu cầu user mua lại vì enrollment ID `21` vẫn là `completed`.

Việc thấy **Start learning** đã được xác nhận. Việc nhấn nút và mở Lesson chưa được thực hiện trước khi chuyển máy.

## 12. Trạng thái tại điểm dừng

```text
Tutor monetization = wc
Course 16          = paid
Product 28         = publish, 500000 VND
Course product     = 28
Enrollment 21      = completed
WooCommerce Orders = 0
```

Chưa thực hiện:

- Chưa nhấn **Start learning** để kiểm tra Lesson sau khi chuyển engine.
- Chưa Add to cart bằng khách mới.
- Chưa Checkout.
- Chưa tạo WooCommerce Order.
- Chưa quan sát Order chuyển thành enrollment.
- Chưa cấu hình hoặc thử payment gateway.

## 13. Bước tiếp tục trên máy mới

Sau khi làm đúng `docs/chuyen-du-an-sang-may-khac.md`:

1. Đăng nhập bằng `hocvien_bai2`.
2. Mở Course ID `16`.
3. Nhấn **Start learning**.
4. Xác nhận Lesson ID `20` mở được và không yêu cầu mua lại.
5. Đăng xuất hoặc dùng cửa sổ ẩn danh.
6. Dùng một tài khoản mới, chưa có enrollment, để bắt đầu thí nghiệm Add to cart → Checkout → Order → Enrollment.

Không dùng `hocvien_bai2` cho thí nghiệm mua mới vì user này đã có enrollment `completed`.

## 14. Điều cần ghi nhớ

1. WooCommerce liên kết Page hệ thống bằng Page ID trong `wp_options`.
2. Hai Page cùng tiêu đề vẫn là hai đối tượng khác nhau nếu ID khác nhau.
3. Shop có thể có `post_content` rỗng vì được render như product archive động.
4. Product là custom post type `product` trong `wp_posts`.
5. Giá và thuộc tính Product nằm trong `wp_postmeta`.
6. `wp_wc_product_meta_lookup` hỗ trợ truy vấn nhanh và được WooCommerce đồng bộ.
7. Tutor chọn WooCommerce bằng `monetize_by = wc`.
8. `_tutor_course_product_id` nối Course với Product.
9. Course chuyển sang Paid không xóa enrollment `completed` đã có.
10. Chỉ việc Course và Product đã liên kết chưa đủ để chứng minh luồng mua hàng; cần tiếp tục quan sát Order và enrollment ở bài sau.

## 15. Câu hỏi ôn tập

1. Vì sao Page Cart của WooCommerce có slug `cart-2`?
2. WooCommerce dùng option nào để biết Page ID của Checkout?
3. Vì sao Page Shop có `post_content` rỗng nhưng vẫn hiển thị được sản phẩm?
4. Cart, Checkout và My account đang dùng ba loại nội dung nào?
5. Product ID `28` được phân biệt với Course ID `16` bằng cột nào?
6. `_regular_price` khác `_price` như thế nào?
7. Vì sao Simple Product có `min_price = max_price`?
8. Meta key nào liên kết Course ID `16` với Product ID `28`?
9. Vì sao `hocvien_bai2` vẫn thấy **Start learning** sau khi Course chuyển sang Paid?
10. Ta còn phải quan sát dữ liệu nào để chứng minh mua Product tạo enrollment?
