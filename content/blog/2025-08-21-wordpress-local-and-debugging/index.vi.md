---
title: WordPress Local and Debugging
description: A powerful and the most popular content management system (CMS).
date: 2025-08-21 09:00:00 +0700
categories: [Web]
tags: [cms, php, wordpress]
weight: 1
images: []
featuredImage: "app.png"

lightgallery: true

toc:
  auto: false
---

---

**WordPress** là một hệ thống quản lý nội dung (CMS) mạnh mẽ và phổ biến nhất cho phép bạn tạo, quản lý và tùy chỉnh các trang web và blog một cách dễ dàng. Nó là open-source CMS, được xây dựng trên PHP và sử dụng cơ sở dữ liệu MySQL hoặc Maria DB.
- Ra đời năm 2003, ban đầu chỉ để viết blog, sau đó phát triển thành nền tảng tạo website, cửa hàng online, diễn đàn, landing page…
- Hiện nay, hơn 40% website trên thế giới chạy bằng WordPress.

Có 2 phiên bản WordPress:
1. WordPress.com
- Dịch vụ hosting do Automattic cung cấp
- Bạn chỉ cần đăng ký tài khoản, không phải cài đặt
- Hạn chế tùy chỉnh, muốn nâng cao thì phải trả phí

2. WordPress.org
- Mã nguồn mở, bạn tự tải về và cài đặt lên hosting/server riêng
- Tùy chỉnh toàn diện, có thể cài plugin, theme, viết code, tạo website theo ý mình

## Ecosystem
- Core: CMS chính
- Plugin: Một phần mềm bổ sung có thể được cài đặt trên trang web WordPress để mở rộng chức năng của nó và thêm các tính năng mới
- Themes: Một phần mềm bổ sung đại diện cho sự xuất hiện trực quan và bố cục của một trang web WordPress

---

## Why WordPress Hacking?
[State of WordPress Security in 2024](https://patchstack.com/whitepaper/state-of-wordpress-security-in-2024)

### Most Popular
- Hiện tại hơn 40% website trên toàn thế giới chạy WordPress
- Nghĩa là hacker chỉ cần tìm ra một lỗ hổng phổ biến => có thể khai thác hàng triệu site cùng lúc
- Giống như `“cá nhiều thì chài ở đó”`

### Plugin and Theme
- WordPress Core đã được xem xét trong 1 thời gian dài bởi hàng ngàn **developers** và **researchers**. Kết quả là rất khó để kẻ tấn công có thể đột nhập vào
- Tuy nhiên, có hàng chục ngàn plugin và theme từ nhiều nguồn, chất lượng không đồng đều được sử dụng
- Nhiều plugin code bảo mật kém, không còn được cập nhật. Hacker chỉ cần scan plugin/theme để tìm version lỗi thời, rồi khai thác

---

## Setup WordPress for Hacking
Có rất nhiều cách để setup WorkPress, tìm kiếm bằng **Google** có thể cung cấp nhiều bài viết về nó. Ở đây tôi sẽ setup trên máy ảo **Ubuntu (22.04)**:
- Không ảnh hưởng đến các dịch vụ của máy thật
- WordPress tương đối nhẹ để sử dụng trên máy ảo

### Install and configure WordPress
#### Install Dependencies
Cài toàn bộ stack cần thiết để chạy WordPress (web server + database + PHP + các extension quan trọng).
```shell
sudo apt install apache2 \
                 ghostscript \
                 libapache2-mod-php \
                 mysql-server \
                 php \
                 php-bcmath \
                 php-curl \
                 php-imagick \
                 php-intl \
                 php-json \
                 php-mbstring \
                 php-mysql \
                 php-xml \
                 php-zip
```

#### Install WordPress

Tải và cài mã nguồn WordPress vào thư mục web.
```shell
# Tạo folder để lưu trữ mã nguồn website
sudo mkdir -p /srv/www

# Đổi owner thành www-data, đây là user mặc định của Apache/Nginx để chạy web server
# Khi Apache/NGINX khởi động, nó không chạy bằng root (nguy hiểm vì có toàn quyền hệ thống), mà sẽ drop xuống chạy dưới quyền www-data.
sudo chown www-data: /srv/www

# Tải gói WordPress mới nhất từ trang chính thức
# Giải nén vao thưc mục /srv/www được tạo ở trên
curl https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
```

Có thể tải phiên bản cụ thể bằng:

```shell
curl https://wordpress.org/wordpress-6.6.2.tar.gz | sudo -u www-data tar zx -C /srv/www
```

> Cài từ `wordpress.org` là cách chuẩn nhất và ít rủi ro:
> - Ubuntu có sẵn package **wordpress** trong repo. Nhưng nó thường cũ hơn nhiều so với bản chính thức trên `wordpress.org`
> - Cộng đồng WordPress chỉ hỗ trợ khi bạn dùng bản chính thức từ `wordpress.org`, vì nếu bạn gặp lỗi do package của Ubuntu thì họ không giải quyết được.
{: .prompt-info }

#### Configure Apache for WordPress
Tạo file `/etc/apache2/sites-available/wordpress.conf` cấu hình Apache cho WordPress:

```text 
# Lắng nghe trên tất cả địa chỉ IP (*) tại port 80 (HTTP).
<VirtualHost *:80>  

    # Thư mục gốc của website (nơi chứa mã nguồn WordPress).
    DocumentRoot /srv/www/wordpress 
    <Directory /srv/www/wordpress>
        Options FollowSymLinks

        # Cho phép file .htaccess ghi đè một số thiết lập liên quan đến bảo mật và URL rewriting.
        AllowOverride Limit Options FileInfo 

        # Khi truy cập thư mục, Apache sẽ tìm file index.php để hiển thị.
        DirectoryIndex index.php 

        # Cho phép tất cả mọi người truy cập (cần cho website public).
        Require all granted 
    </Directory>

    # Riêng thư mục wp-content (chứa plugin, theme, upload) cũng cho phép truy cập.
    <Directory /srv/www/wordpress/wp-content> 
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```

**Kích hoạt site WordPress**:

```shell
sudo a2ensite wordpress
```
- **a2ensite** = `enable site config` (tạo symlink từ `/etc/apache2/sites-available/wordpress.conf` sang `/etc/apache2/sites-enabled/`)
- Sau đó site wordpress sẽ được Apache load khi restart.

**Kích hoạt module rewrite**:

```shell
sudo a2enmod rewrite
```
- WordPress cần module mod_rewrite để hỗ trợ **Pretty Permalinks** (ví dụ `/blog/hello-world/` thay vì `?p=123`)

**Vô hiệu hóa site mặc định (Option)**:

```shell
sudo a2dissite 000-default
```
- Apache mặc định có site `000-default.conf` (chỉ hiển thị "It works!")
- Nếu không disable, site mặc định có thể chiếm port `80` trước WordPress

👉 Hoặc nếu không muốn disable, có thể thêm ServerName vào VirtualHost WordPress để Apache phân biệt domain.

```text
<VirtualHost *:80>
    ServerName mywordpress.local
    DocumentRoot /srv/www/wordpress
    ...
</VirtualHost>
```
Muốn chạy được thì phải cấu hình DNS hoặc sửa file hosts:
- Linux/Mac: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`

**Reload Apache**:

```shell
sudo service apache2 reload
```
- Tải lại cấu hình mới mà không cần dừng server

#### Configure database
WordPress cần một cơ sở dữ liệu (MySQL/MariaDB) để lưu:
- Bài viết, trang, menu
- Người dùng, mật khẩu (hash)
- Cấu hình site
- Plugins, themes settings

**Mở MySQL với quyền root**:

```shell
sudo mysql -u root
```

**Cấu hình MySQL**:

```sql
-- Tạo database wordpress 
mysql> CREATE DATABASE wordpress;
Query OK, 1 row affected (0,00 sec)

-- Tạo user riêng cho wordpress
mysql> CREATE USER wordpress@localhost IDENTIFIED BY '<your-password>';
Query OK, 1 row affected (0,00 sec)

-- Cấp quyền cho user WordPress
mysql> GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER ON wordpress.* TO wordpress@localhost;
Query OK, 1 row affected (0,00 sec)

-- Làm mới lại quyền
mysql> FLUSH PRIVILEGES;
Query OK, 1 row affected (0,00 sec)

-- Thoát
mysql> quit
Bye
```

**Khởi động lại dịch vụ**:

```shell
sudo service mysql start
```

#### Configure WordPress to connect to the database
**Copy file cấu hình mẫu**:

```shell
sudo -u www-data cp /srv/www/wordpress/wp-config-sample.php /srv/www/wordpress/wp-config.php
```
- WordPress khi tải về có sẵn file wp-config-sample.php
- Bạn copy nó thành wp-config.php để làm file cấu hình chính cho site

**Chỉnh sửa thông tin database**:

```shell
sudo -u www-data sed -i 's/database_name_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/username_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/password_here/<your-password>/' /srv/www/wordpress/wp-config.php
```
- Đây là 3 dòng thay thế text trong file `wp-config.php`:
    - `database_name_here` => `wordpress` (tên DB đã tạo ở bước trước)
    - `username_here` => `wordpress` (user MySQL đã tạo)
    - `password_here` => `<your-password>` (password đã set)
- Khi Apache chạy WordPress, nó sẽ đọc file này để biết dùng user nào + DB nào để kết nối MySQL

**Thêm Secret Keys và Salts**:

```shell
sudo -u www-data nano /srv/www/wordpress/wp-config.php
```

👉 Đây là chuỗi bí mật mà WordPress dùng để:
- Mã hóa session cookies
- Sinh ra token (nonce) chống CSRF
- Ngăn chặn tấn công kiểu “known secret” (ví dụ kẻ tấn công đoán được secret mặc định để giả mạo login)

```php
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );
```

Thay đổi nó bằng các chuỗi random sau: [https://api.wordpress.org/secret-key/1.1/salt/](https://api.wordpress.org/secret-key/1.1/salt/)

#### Configure WordPress
Truy cập [http://localhost](http://localhost), đặt **site title**, **username**, **password**, **email** cho user `admin`

![Configure WordPress](https://ubuntucommunity.s3.us-east-2.amazonaws.com/original/2X/e/ebe4d6066e0c32f14beca85ffd53e5915a4ab278.png)

### Setup Debug on VSCode
Để hiểu được luồng của trang web chạy và xử lí ở mỗi request, ta cần phải debug nó.

#### Add PHP Debug extension on VS Code
Vào **Extensions (Ctrl+Shift+X)** => tìm **PHP Debug (by Felix Becker)** => **Install**.
Extension này kết nối với **Xdebug**.

#### Install Xdebug on Ubuntu

```shell
sudo apt install php-xdebug -y
```

Kiểm tra cài chưa:

```shell
php -v
```
Nếu có dòng `with Xdebug v3.x.x` nghĩa là đã OK.

#### Configure Xdebug

```shell
sudo nano /etc/php/<version>/apache2/php.ini
```

Thêm vào cuối file

```ini
zend_extension=xdebug.so
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
```

Port mặc định mới của Xdebug v3 là **9003**.

Sau đó restart Apache:

```shell
sudo systemctl restart apache2
```

#### Configure VSCode `launch.json`
Mở thư mục chứa trang web wordpress bằng VSCode

```
code /srv/www/wordpress
```

Trong **VS Code** => **Run and Debug (Ctrl+Shift+D)** => tạo file `launch.json` với nội dung:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003,
      "pathMappings": {
        "/srv/www/wordpress": "${workspaceFolder}"
      }
    }
  ]
}
```

`pathMappings`: ánh xạ thư mục trong **container/server** (`/srv/www/wordpress`) về thư mục **local** của trong VS Code.

---

Như vậy là đã hoàn thành setup WordPress trên local kết hợp Debug nó trên VSCode.

---

## Extend
### Required Version
Mỗi phiên bản WordPress thường đi kèm 1 phiên bản PHP thích hợp. Nên kiểm tra tính tương thích này để quá trình setup không bị lỗi.

### WordPress Auto Update 
Từ WordPress **3.7 trở đi** (2013), WordPress có cơ chế tự động cập nhật (**automatic background updates**) cho:
- Bản vá bảo mật (security release).
- Bản bảo trì (maintenance release).
- Không tự động update major version (trừ khi bật rõ ràng).

Hệ thống kiểm tra với server **api.wordpress.org** để xem có bản vá nào mới sẽ tự động nâng cấp background lên. Việc này diễn ra rất nhanh sau khi setup, nên bạn thấy code đổi khác so với source gốc. 

Quá trình setup diễn ra khi truy cập **/wp-admin/install.php**

Để vô hiệu hóa cơ chế cập nhật tự động này, thêm dòng sau vào `wp-config.php`:

```php
define( 'WP_AUTO_UPDATE_CORE', false );
```

---