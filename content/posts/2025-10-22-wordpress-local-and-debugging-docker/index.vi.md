---
title: WordPress Local and Debugging in Docker
description: A powerful and the most popular content management system (CMS).
date: 2025-10-22 15:00:00 +0700
categories: [Web]
tags: [cms, php, wordpress, docker]
images: ["app.png"]
featuredImage: "app.png"

lightgallery: true

toc:
  auto: false
---

<!--more-->

**Tiếp nối** phần hướng dẫn **cài đặt WordPress trên môi trường local** ở [bài viết trước](https://w41bu1.github.io/2025-08-21-wordpress-local-and-debugging/), bài này sẽ hướng dẫn bạn **thiết lập WordPress bằng Docker** — một cách hiện đại, linh hoạt và dễ chia sẻ hơn.

## Why Use Docker?

Docker cho phép bạn đóng gói toàn bộ môi trường chạy WordPress (PHP, MySQL, web server và mã nguồn) vào các container độc lập. Cách làm này mang lại nhiều lợi ích:

- **Đồng nhất môi trường**: Chạy giống nhau trên mọi máy, tránh lỗi “chạy được trên máy tôi”.
- **Dễ cài đặt và tái tạo**: Chỉ cần vài lệnh để khởi tạo hoặc reset toàn bộ hệ thống.
- **Cách ly và an toàn**: Mỗi container là một môi trường riêng, tránh xung đột phần mềm.
- **Thuận tiện cho phát triển và debug**: Có thể bật Xdebug, theo dõi log, hoặc chỉnh PHP mà không ảnh hưởng đến hệ thống chính.
- **Dễ chia sẻ**: Chỉ cần gửi file `docker-compose.yml` và các file cấu hình, người khác có thể khởi chạy cùng môi trường giống bạn — không cần cài đặt thủ công.

Docker là **giải pháp tối ưu** để phát triển, thử nghiệm và cộng tác trên các dự án WordPress, đặc biệt hữu ích cho lập trình viên, nhóm làm việc hoặc người làm pentest muốn có môi trường nhất quán, dễ tái sử dụng.

## Setup WordPress for Hacker in Docker

### Prerequisites

#### Docker
Trước tiên, bạn cần cài đặt **Docker**. Công cụ này hỗ trợ trên nhiều hệ điều hành, mỗi nền tảng có cách cài đặt riêng.  
Xem hướng dẫn tại: [https://www.docker.com/get-started/](https://www.docker.com/get-started/)

#### Docker Compose
Thay vì chạy từng container thủ công bằng `docker run`, ta dùng **Docker Compose** để định nghĩa và quản lý nhiều container liên kết chỉ bằng **một file cấu hình duy nhất** (`docker-compose.yml`).

Với WordPress, ta cần:
- Một container **MySQL** để lưu cơ sở dữ liệu.
- Một container **WordPress (PHP + web server)** để chạy mã nguồn.

Nếu làm thủ công, bạn sẽ phải nhập nhiều lệnh và cấu hình phức tạp.  
Với Docker Compose, tất cả nằm trong một file YAML giúp:

- **Dễ cài đặt**: `docker-compose up -d` là có đủ WordPress + MySQL.  
- **Dễ chia sẻ**: Người khác chỉ cần cùng file `docker-compose.yml` là chạy được.  
- **Dễ mở rộng**: Thêm phpMyAdmin, Nginx, hoặc Xdebug chỉ cần vài dòng.

Xem hướng dẫn cài Docker Compose tại: [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

### Installing WordPress with Docker Compose and Xdebug

Phần này hướng dẫn cài **WordPress với Docker Compose**, tích hợp **Xdebug** để debug trực tiếp trong **VS Code**.  
Cách này giúp bạn có môi trường phát triển đầy đủ, dễ gỡ lỗi và mở rộng.

**Cấu trúc thư mục:**

```sh
.
├── wordpress
├── docker-compose.yml
├── Dockerfile
├── php.ini
└── .vscode
    └── launch.json
````

### Setup Environment

Tạo thư mục mới để chứa toàn bộ cấu hình:

```sh
mkdir wordpress-docker && cd wordpress-docker
```

### Create Dockerfile

Image WordPress mặc định **chưa có Xdebug**, nên ta mở rộng bằng `Dockerfile`.

```sh
nano Dockerfile
```

**Nội dung:**

```dockerfile
FROM wordpress:latest

RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

ARG UID=1000
ARG GID=1000

RUN groupmod -g ${GID} www-data && \
    usermod -u ${UID} -g ${GID} www-data
```

### Create docker-compose.yml

Tạo file để cấu hình toàn bộ hệ thống:

```sh
nano docker-compose.yml
```

**Nội dung:**

```yaml
services:
  db:
    image: mysql:latest
    container_name: wp_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    build: .
    container_name: wp_app
    depends_on:
      - db
    ports:
      - "80:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./wordpress:/var/www/html
      - ./php.ini:/usr/local/etc/php/conf.d/php.ini

volumes:
  db_data:
```

> [!INFO]
> `volumes:` dùng để **lưu trữ dữ liệu và đồng bộ mã nguồn** giữa máy thật và container.

**Giải thích:**

* **`db_data:/var/lib/mysql`**
  Lưu dữ liệu MySQL bền vững, không mất khi xóa container.

* **`./wordpress:/var/www/html`**
  Đồng bộ mã nguồn WordPress giữa máy và container, giúp sửa code trực tiếp.

* **`./php.ini:/usr/local/etc/php/conf.d/php.ini`**
  Gắn file cấu hình PHP/Xdebug, dễ tùy chỉnh mà không cần rebuild image.

### Create php.ini

Kích hoạt Xdebug và cấu hình PHP.

```sh
nano php.ini
```

**Nội dung:**

```ini
zend_extension=xdebug
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=172.17.0.1
xdebug.client_port=9003
xdebug.log_level=0

upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 128M
max_execution_time = 300
max_input_time = 300
```

**Giải thích:**

* Các dòng `xdebug.*` giúp kết nối với VS Code khi debug.
* Các dòng sau tăng giới hạn upload và tài nguyên PHP.

### Create VS Code Launch Configuration

Tạo file cấu hình VS Code để kết nối với Xdebug:

```sh
mkdir .vscode && nano .vscode/launch.json
```

**Nội dung:**

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
        "/var/www/html": "${workspaceFolder}/wordpress"
      },
      "log": true
    }
  ]
}
```

### Install VS Code Extension

Mở VS Code → **Extensions (Ctrl + Shift + X)** → Cài đặt:

> **PHP Debug (by Xdebug)**

### Run Docker Compose

Khởi động hệ thống:

```sh
docker-compose up -d
```

Sau khi khởi chạy, truy cập **[http://localhost](http://localhost)** để cài đặt WordPress.
Giờ bạn có thể đặt breakpoint trong file PHP và debug trực tiếp trên VS Code.

