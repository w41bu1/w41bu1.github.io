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

**Continuing** from the previous guide on **installing WordPress in a local environment** ([see here](https://w41bu1.github.io/2025-08-21-wordpress-local-and-debugging/)), this post will show you how to **set up WordPress with Docker** — a modern, flexible, and easily shareable approach.

## Why Use Docker?

Docker allows you to package the entire WordPress environment (PHP, MySQL, web server, and source code) into independent containers. This brings several key advantages:

- **Environment consistency**: Works the same everywhere, eliminating “works on my machine” issues.  
- **Easy setup and reset**: Spin up or rebuild your environment in just a few commands.  
- **Isolation and security**: Each container runs independently, avoiding software conflicts.  
- **Convenient development and debugging**: Enable Xdebug, monitor logs, or tweak PHP settings without affecting your main system.  
- **Easy sharing**: Just share the `docker-compose.yml` file — others can launch the exact same setup without manual installation.

Docker is an **optimal solution** for developing, testing, and collaborating on WordPress projects — especially useful for developers, teams, or penetration testers who need a consistent, reproducible environment.

## Setup WordPress for Hacker in Docker

### Prerequisites

#### Docker
First, install **Docker**. It’s available on all major operating systems, with installation steps differing slightly per platform.  
See the official guide: [https://www.docker.com/get-started/](https://www.docker.com/get-started/)

#### Docker Compose
Instead of manually starting containers with `docker run`, use **Docker Compose** — a tool that defines and manages multiple linked containers using **a single configuration file** (`docker-compose.yml`).

For WordPress, you’ll need:
- A **MySQL** container for the database.
- A **WordPress (PHP + web server)** container for the site itself.

Manually setting this up involves complex commands and configurations.  
With Docker Compose, everything is neatly handled in one YAML file:

- **Easy setup**: Run `docker-compose up -d` to get WordPress + MySQL instantly.  
- **Easy sharing**: Anyone can use your `docker-compose.yml` to replicate your setup.  
- **Easy expansion**: Add phpMyAdmin, Nginx, or Xdebug with just a few extra lines.

Install instructions: [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

### Installing WordPress with Docker Compose and Xdebug

This section shows how to install **WordPress using Docker Compose**, with **Xdebug integration** for debugging directly in **VS Code**.  
This gives you a complete development setup that’s easy to debug and extend.

**Folder structure:**

```sh
.
├── wordpress
├── docker-compose.yml
├── Dockerfile
├── php.ini
└── .vscode
    └── launch.json
```

### Setup Environment

Create a new folder to store all configuration files:

```sh
mkdir wordpress-docker && cd wordpress-docker
```

### Create Dockerfile

The default WordPress image **does not include Xdebug**, so we’ll extend it with a `Dockerfile`.

```sh
nano Dockerfile
```

**Content:**

```dockerfile
FROM wordpress:latest

# Install Xdebug
RUN pecl install xdebug && docker-php-ext-enable xdebug
```

### Create docker-compose.yml

Now create the file that configures the full environment:

```sh
nano docker-compose.yml
```

**Content:**

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
> The `volumes:` section handles **data persistence and source code synchronization** between your host machine and containers.

**Explanation:**

* **`db_data:/var/lib/mysql`**  
  Stores MySQL data persistently — your database won’t be lost if the container is removed.

* **`./wordpress:/var/www/html`**  
  Syncs WordPress source code between your machine and the container, allowing live editing.

* **`./php.ini:/usr/local/etc/php/conf.d/php.ini`**  
  Mounts your PHP/Xdebug config file for easy customization without rebuilding the image.

### Create php.ini

Enable Xdebug and adjust PHP settings.

```sh
nano php.ini
```

**Content:**

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

**Explanation:**

- The `xdebug.*` lines enable debugging in VS Code.
- The remaining lines increase upload and memory limits for PHP.

### Create VS Code Launch Configuration

Create the VS Code config file to connect with Xdebug:

```sh
mkdir .vscode && nano .vscode/launch.json
```

**Content:**

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

Open VS Code → **Extensions (Ctrl + Shift + X)** → Search and install:

> **PHP Debug (by Xdebug)**

### Run Docker Compose

Start the environment:

```sh
docker-compose up -d
```

After the containers start, visit **[http://localhost](http://localhost)** to complete the WordPress installation.  
You can now set breakpoints in your PHP code and debug directly in VS Code.
