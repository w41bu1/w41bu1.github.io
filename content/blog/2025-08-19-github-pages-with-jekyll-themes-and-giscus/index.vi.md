---
title: Github Pages with Jekyll themes and Giscus
description: How to create your personal blog using Github Pages with "comment" feature
date: 2025-08-19 01:10:00 +0700
categories: [Web]
tags: [blog, jekyll, giscus, github]
weight: 1
images: []
featuredImage: "app.png"

lightgallery: true

toc:
  auto: false
---

---

Khi bắt đầu tạo blog cá nhân, có ba công cụ bạn cần biết: GitHub Pages, Jekyll và Giscus.
- **Github Pages**: Dịch vụ miễn phí từ GitHub, cho phép bạn triển khai website tĩnh trực tiếp từ repository. Chỉ cần push code lên GitHub, blog của bạn sẽ tự động xuất hiện trên Internet mà không cần server riêng.

- **Jekyll**: Một static site generator tích hợp sẵn với GitHub Pages. Jekyll giúp bạn dễ dàng tạo blog từ các file Markdown, tổ chức nội dung bằng template và theme sẵn có.

- **Giscus**: Một hệ thống bình luận hiện đại dựa trên GitHub Discussions. Thay vì phải dùng dịch vụ ngoài như Disqus, bạn có thể tận dụng GitHub để quản lý comment, vừa gọn nhẹ vừa thân thiện với developer.

---

## Requirements
- Tài khoản [**Github**](https://github.com/)
- Kiến thức cơ bản về [**Markdown**](https://markdownlivepreview.com/)

---

## Creating a Site Repository
Ở đây, tôi sử dụng [**Chirpy themes**](https://chirpy.cotes.page/), đây là theme khá nổi tiếng dành cho GitHub Pages, được tối ưu cho việc viết blog cá nhân hoặc technical blog.

**Các bước thực hiện**:  
- Đăng nhập vào [**Github**](https://github.com/) và di chuyển đến [**starter**](https://github.com/cotes2020/chirpy-starter)
- Thay vì Fork, ta click <kbd>Use this template</kbd> và chọn <kbd>Create a new repository</kbd> để tự động tạo Site Repository.
- Đặt tên cho Repository mới `<username>.github.io`, thay đổi `<username>` thành Github username của bạn.

---

## Setting up the Environment
Có 2 lý do chính bạn nên setup môi trường trên máy local khi phát triển blog:  
- Sau khi push code lên Repository, GitHub Actions sẽ mất một khoảng thời gian để chạy rồi mới build và render ra GitHub Pages.  
- Khi dev trực tiếp trên local, bạn có thể quan sát kết quả ngay lập tức, nhanh hơn và thuận tiện hơn.  

### Using Dev Containers (Recommended for Windows)
**Dev Containers** cung cấp một môi trường cô lập bằng Docker, giúp tránh xung đột với hệ thống và đảm bảo mọi dependency đều được quản lý trong container.  

**Các bước thực hiện**:  
1. Cài đặt Docker:  
   - Trên Windows/macOS: cài [Docker Desktop](https://www.docker.com/products/docker-desktop/).  
   - Trên Linux: cài [Docker Engine](https://docs.docker.com/engine/install/).  
2. Cài [VS Code](https://code.visualstudio.com/) và extension [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers).  
3. Clone repository:  
   - Nếu dùng Docker Desktop: mở VS Code và [clone repo trong container volume](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-a-git-repository-or-github-pr-in-an-isolated-container-volume).  
   - Nếu dùng Docker Engine: clone repo về local, sau đó [mở bằng container](https://code.visualstudio.com/docs/devcontainers/containers#_quick-start-open-an-existing-folder-in-a-container) trong VS Code.  
4. Chờ quá trình thiết lập Dev Containers hoàn tất.  

### Setting up Natively (Recommended for Unix-like OS)
Với các hệ điều hành dạng Unix-like (Linux, macOS), bạn có thể setup môi trường trực tiếp (native) để đạt hiệu năng tốt nhất. Ngoài ra, vẫn có thể dùng Dev Containers như một lựa chọn thay thế.  

**Các bước thực hiện**:  
1. Làm theo [hướng dẫn cài đặt Jekyll](https://jekyllrb.com/docs/installation/) và đảm bảo đã cài đặt [Git](https://git-scm.com/).  
2. Clone repository về máy local.  
3. Nếu bạn fork theme, cài [Node.js](https://nodejs.org/) và chạy `bash tools/init.sh` trong thư mục gốc để khởi tạo repo.  
4. Chạy lệnh dưới đây ở thư mục gốc để toàn bộ gem sẽ được cài vào `./vendor/bundle/` trong project, không cần **sudo** và không đụng gì đến `/var/lib/gems..`

```shell
bundle config set --local path 'vendor/bundle'
bundle install
```

---

## Usage
### Start the Jekyll Server
Để chạy site trên local, sử dụng lệnh dưới đây:

```shell
bundle exec jekyll s
```

### Configuration
Một số biến cần cấu hình trong `_config.yml`, bao gồm:
- `lang`: Set language cho website của bạn
- `url`: Trỏ đến website của bạn
- `title`: Title chính, nằm dưới avatar
- `tagline`: Subtitle, mô tả trang web
- `avatar`: hỗ trợ local và CORS resources, có thể sử dụng `gif`

### Comment feature via Giscus
Ta sử dụng [**Giscus**](https://giscus.app) làm hệ thống comment cho website, ngoài ra còn các option khác như: [**Disqus**](https://disqus.com/), [**Utterances**](https://utteranc.es/), tất cả đều miễn phí.

**Các bước thực hiện**:  
1. Cài đặt [**giscus**](https://github.com/apps/giscus) vào Github.
2. Vào **Settings** của repository chọn **General** và bật **Discussions** để giscus lưu comments vào **Discussions**.
3. Nhập repository `<username>/<username>.github.io`, thông báo màu xanh lá xuất hiện khi các tiêu chí ở trên được đáp ứng. 
4. Chọn thể loại discussion và chủ đề cho website.
5. Cấu hình giscus trong `_config.yml`
    - `provider`: chọn giscus
    - `giscus`: mapping các biến tương ứng đã làm ở [**giscus**](https://github.com/apps/giscus) vào biến `giscus`.

### Customize the Favicon
Tạo **favicon** riêng cho website của bạn, không phải sử dụng **favicon** mặc định của themes

**Các bước thực hiện**:  
1. Truy cập [**Favicon Genarate**](https://www.favicon-generator.org/) 
2. Click <kbd>Browse</kbd> chọn favicon cần tạo, sau đó click <kbd>Create Favicon</kbd> để tạo.
3. Click `Download the generated favicon` để tải các file chứa các favicon về.
4. Giải nén file vừa tải, xóa 2 file dưới đây:
    - browserconfig.xml
    - site.webmanifest
5. Copy toàn bộ file còn lại và paste vào `assets/img/favicons`(nếu chưa có thì tạo).

---

## Extends
### Remove meta tag in Footer
Có thể xóa dòng `Using the Chirpy theme for Jekyll.` bằng các bước sau:
1. Tạo file theo đường dẫn sau `_data/locales/en.yml`.
2. Copy nội dung [**en.yml**](https://raw.githubusercontent.com/cotes2020/jekyll-theme-chirpy/refs/heads/master/_data/locales/en.yml) và dán vào file vừa tạo.
3. Sửa nội dung biến `meta` thành `""`

### Write a post
Tham khảo [**rule**](https://chirpy.cotes.page/posts/write-a-new-post/) để viết blog của themes và sử dụng [**Live Preview**](https://markdownlivepreview.com/) để quan sát

---