# Bypass LFI Forced .php Suffix Via pearcmd


<!--more-->

Xin chào. Sau khi phân tích 10 CVE liên quan đến LFI trên các plugin WordPress, tôi nhận thấy một rào cản chung: nhiều vector khai thác bị giới hạn bởi yêu cầu **phải có hậu tố `.php`**. Điều này làm giảm đáng kể khả năng khai thác. Trong quá trình tìm hiểu, tôi tìm thấy bài viết [docker-php-include-getshell](https://www.leavesongs.com/PENETRATION/docker-php-include-getshell.html#0x06-pearcmdphp). Bài đó mô tả cách **bypass ràng buộc `.php`** bằng cách lợi dụng file `pearcmd.php` vốn nằm trong bộ công cụ **PECL**/**PEAR** của PHP và có sẵn trong môi trường WordPress được triển khai trên Docker - một mẹo rất thực tế cho kịch bản không cho upload file.

## PEAR và PECL là gì?
- **PECL (PHP Extension Community Library)**: công cụ dòng lệnh để cài và quản lý extension PHP.  
- **PEAR (PHP Extension and Application Repository)**: thư viện nền tảng cho PECL.

Trước PHP `7.3`, PEAR/PECL thường được cài mặc định.  
Từ PHP `7.4` trở đi, cần biên dịch PHP với `--with-pear` để có chúng.

Tuy nhiên, trong các Docker image PHP chính thức, PEAR/PECL **vẫn thường được cài sẵn**, nằm ở `/usr/local/lib/php`:

```sh
root@e182501c47c4:/var/www/html# ls /usr/local/lib/php
Archive  Console  OS  PEAR  PEAR.php  Structures  System.php  XML  build  data  doc  extensions  pearcmd.php  peclcmd.php  test
```

## `pearcmd.php` và `register_argc_argv`
`pearcmd.php` là một script PHP thiết kế để chạy ở chế độ dòng lệnh, ví dụ:

```sh
php /usr/local/lib/php/pearcmd.php install somepackage
```

Nó xử lý các tham số từ `$argv` và `$argc`. Khi chạy như CLI thì dữ liệu này rõ ràng. Nếu file này được include trong bối cảnh web (do LFI), logic CLI của nó có thể bị lợi dụng.

Điểm then chốt là cấu hình `register_argc_argv`. Nếu `register_argc_argv = On`, PHP sẽ tạo:
- `$argc`
- `$argv`
- `$_SERVER['argv']`

![argv](argv.png "`argv` trong cấu hình PHP")

Khi thiết lập WordPress trên Docker, `register_argc_argv` thường bật mặc định. Vấn đề đặt ra: khi PHP chạy dưới SAPI web (FPM/Apache) và không phải CLI, `$argv` lấy dữ liệu từ đâu?

## Analysis PHP Source Code
Trong PHP core có logic như sau:

```c
if (PG(register_argc_argv)) {
    if (SG(request_info).argc) {
        ...
    } else {
        php_build_argv(SG(request_info).query_string, &PG(http_globals)[TRACK_VARS_SERVER]);
    }
}
```

Nếu không có `argc` (không chạy CLI), PHP gọi `php_build_argv` với `SG(request_info).query_string` - tức **query string** của URL. Ví dụ:

```
http://example.com/index.php?a=b&c=d
```

→ `query_string = "a=b&c=d"`

PHP sẽ dùng query string này để tạo biến `argv`, do đó `$_SERVER['argv']` có thể bị ảnh hưởng bởi query string.

**Hậu quả**:

Khi `pearcmd.php` được include qua LFI trong môi trường web mà `$_SERVER['argv']` được sinh từ query string, attacker có thể điều khiển các tham số dòng lệnh mà `pearcmd.php` đọc được. Do đó, chức năng dòng lệnh của PEAR/PECL có thể bị lợi dụng qua web để thực hiện hành động không mong muốn.

## RFC3875 Explain

RFC3875 (CGI spec) định nghĩa một dạng **"indexed" HTTP query** - tức là chuỗi truy vấn không chứa dấu `=` chưa mã hóa và được gửi qua phương thức GET hoặc HEAD.

Khi gặp loại truy vấn này, server **nên** (`SHOULD`) coi phần query-string như một **search-string**, tách thành các **search-word** bằng dấu `+`:

```
search-string = search-word ( "+" search-word )
search-word = 1*schar
```

Sau khi tách, mỗi `search-word` sẽ được **URL-decode**, có thể mã hóa lại theo hệ thống, rồi **thêm vào danh sách đối số dòng lệnh (`argv`)** của chương trình CGI.

Nói ngắn gọn: nếu query-string không có dấu `=` và là yêu cầu GET hoặc HEAD, máy chủ có thể coi các phần ngăn cách bằng `+` như các tham số dòng lệnh và truyền vào `argv`.

**RFC3875** cho phép server biến một query-string "indexed" (GET/HEAD, không có ký tự `=` chưa mã hóa) thành danh sách từ, rồi đưa vào `argv`. Phần mô tả trong tiêu chuẩn như sau:

```
4.4. The Script Command Line

Some systems support a method for supplying an array of strings to
the CGI script. This is only used in the case of an 'indexed' HTTP
query, which is identified by a 'GET' or 'HEAD' request with a URI
query string that does not contain any unencoded "=" characters. For
such a request, the server SHOULD treat the query-string as a
search-string and parse it into words, using the rules

search-string = search-word ( "+" search-word )
search-word = 1*schar
schar = unreserved | escaped | xreserved
xreserved = ";" | "/" | "?" | ":" | "@" | "&" | "=" | "," |
"$"

After parsing, each search-word is URL-decoded, optionally encoded in
a system-defined manner and then added to the command line argument
list.
```

PHP từng có lỗ hổng liên quan ([CVE-2012-1823](https://nvd.nist.gov/vuln/detail/cve-2012-1823)). Hiện nay, PHP xử lý query-string rộng hơn RFC - ngay cả khi query-string có dấu `=`, nó vẫn có thể được đưa vào `$_SERVER['argv']`.

![equal](equal.png "PHP vẫn thêm query-string có dấu `=` vào $_SERVER['argv']")

## `pearcmd` Code Analysis

> [!INFOMATION]
> Trước khi phân tích mã nguồn của `/usr/local/lib/php/pearcmd.php` ta cần setup debug để hiểu được luồng hoạt động của nó
