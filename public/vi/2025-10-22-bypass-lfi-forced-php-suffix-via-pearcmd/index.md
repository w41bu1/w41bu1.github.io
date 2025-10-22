# Bypass LFI Forced .php Suffix Via pearcmd


<!--more-->

Xin chào. Sau khi phân tích 10 CVE liên quan đến LFI trên các plugin WordPress, tôi nhận thấy một rào cản chung: nhiều vector bị bó hẹp bởi yêu cầu **phải có hậu tố `.php`**, điều này làm giảm đáng kể khả năng khai thác và rất khó chịu. Sau một thời gian tìm tòi, tôi tình cờ tìm thấy một bài viết hữu ích: [docker-php-include-getshell](https://www.leavesongs.com/PENETRATION/docker-php-include-getshell.html#0x06-pearcmdphp). Bài đó mô tả cách **bypass ràng buộc `.php`** bằng cách lợi dụng file `pearcmd.php` vốn nằm trong bộ công cụ **PECL**/**PEAR** của PHP và có sẵn trong môi trường WordPress được triển khai trên Docker - một mẹo rất thực tế cho kịch bản không cho upload file.

## What is PEAR and PECL?
* **PECL (PHP Extension Community Library)**: công cụ dòng lệnh dùng để cài đặt và quản lý extension (mở rộng) PHP.
* **PEAR (PHP Extension and Application Repository)**: là thư viện nền mà PECL dựa vào.

Trước PHP `7.3`, **PECL**/**PEAR** được cài mặc định.
Từ PHP `7.4` trở đi, muốn cài bạn phải biên dịch PHP với tùy chọn `--with-pear`.

👉 Tuy nhiên, trong Docker image PHP, bất kể phiên bản nào, PEAR/PECL vẫn được cài mặc định, nằm ở đường dẫn: `/usr/local/lib/php`

```sh
root@e182501c47c4:/var/www/html# ls /usr/local/lib/php
Archive  Console  OS  PEAR  PEAR.php  Structures  System.php  XML  build  data	doc  extensions  pearcmd.php  peclcmd.php  test
```

## What is `pearcmd.php` and `register_argc_argv`?

`pearcmd.php` là một file PHP có thể chạy như một script dòng lệnh, ví dụ:

```sh
php /usr/local/lib/php/pearcmd.php install somepackage
```

Tức là nó nhận các tham số như `argv`, `argc` để xử lý lệnh.

Bình thường, nó không nằm trong thư mục web, nên không thể bị truy cập qua HTTP, nên an toàn.

Nhưng trong một lỗ hổng **Local File Inclusion (LFI)**, kẻ tấn công có thể ép PHP include file này, dẫn đến việc lợi dụng logic bên trong nó để thực thi lệnh gián tiếp.

Máu chốt nằm ở `register_argc_argv` trong `phpinfo()`. Nếu bật `register_argc_argv = On`, PHP sẽ tạo ra các biến:
* `$argc`
* `$argv`
* `$_SERVER['argv']`

`register_argc_argv=on` mặc định khi setup wordpress trong docker

Khi PHP chạy theo chế độ **CLI** thì bình thường (`php script.php arg1 arg2` → `$argv = ['script.php', 'arg1', 'arg2']`).

Nhưng khi PHP chạy trong chế độ **web server (SAPI = apache, fpm)**, nếu vẫn bật `register_argc_argv`, thì câu hỏi là: `“$argv lấy dữ liệu từ đâu?”`

## Analysis PHP source code

Đoạn mã trong PHP core cho thấy:

```php
if (PG(register_argc_argv)) {
    if (SG(request_info).argc) {
        ...
    } else {
        php_build_argv(SG(request_info).query_string, &PG(http_globals)[TRACK_VARS_SERVER]);
    }
}
```

Tức là nếu không có `argc` (tức là PHP không chạy theo CLI), thì PHP sẽ gọi hàm:

```php
php_build_argv(SG(request_info).query_string, &PG(http_globals)[TRACK_VARS_SERVER]);
```

Mà `SG(request_info).query_string` chính là phần **query string** trong URL, ví dụ:

```http
http://example.com/index.php?a=b&c=d
```

→ `query_string = "a=b&c=d"`

Và PHP sẽ dùng phần này để tạo biến `argv`.

Biến $_SERVER['argv'] bị ảnh hưởng bởi query string

Nên nếu có file pearcmd.php (thiết kế để chạy qua CLI, dùng argv làm tham số),
thì giờ khi include qua web, ta có thể điều khiển argv thông qua query string.
