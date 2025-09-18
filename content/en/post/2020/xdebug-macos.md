---
layout: post
title: Debugging PHP on macOS with VS Code (xdebug)
category: PHP
date: 2018-05-21
tags:
- Visual Studio Code
- debug
- PHP
- xdebug
---
# Debugging PHP on macOS with VS Code (xdebug)

Windows guide: https://zhuanlan.zhihu.com/p/25844268

## Homebrew

Install brew from https://brew.sh or:

```sh
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## PHP7 + nginx + php-fpm + xdebug

### PHP 7

```sh
brew install php@7.1
where php
# /usr/local/opt/php@7.1/bin/php
# /usr/bin/php
```

`php.ini` is typically under `/usr/local/etc/php/7.1` (we’ll need it later).

## Install xdebug

Homebrew’s `php71-xdebug` was removed. Build manually. Download from https://xdebug.org/files/ and compile:

```sh
mkdir ~/tool && cd ~/tool
wget https://xdebug.org/files/xdebug-2.6.0.tgz
tar xvzf xdebug-2.6.0.tgz
cd xdebug-2.6.0
phpize
./configure --enable-xdebug --with-php-config=/usr/local/Cellar/php@7.1/7.1.17/bin/php-config
make && make test
# xdebug-2.6.0/modules/xdebug.so will be generated
```

## Install nginx

```sh
brew install nginx
```

### nginx.conf

Create directories and a minimal config:

```sh
mkdir -p /usr/local/var/logs/nginx
mkdir -p /usr/local/etc/nginx/{sites-available,sites-enabled,conf.d,ssl}
sudo mkdir -p /var/www && sudo chown :staff /var/www && sudo chmod 777 /var/www
```

`/usr/local/etc/nginx/nginx.conf`:

```conf
user root wheel;
worker_processes 1;
error_log /usr/local/var/logs/nginx/error.log debug;
pid /usr/local/var/run/nginx.pid;
events { worker_connections 256; }
http {
  include mime.types;
  default_type application/octet-stream;
  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /usr/local/var/logs/access.log main;
  sendfile on;
  keepalive_timeout 65;
  port_in_redirect off;
  include /usr/local/etc/nginx/sites-enabled/*;
}
```

`/usr/local/etc/nginx/conf.d/php-fpm`:

```conf
location ~ \.php$ {
  try_files $uri =404;
  fastcgi_pass 127.0.0.1:9000;
  fastcgi_index index.php;
  fastcgi_intercept_errors on;
  include /usr/local/etc/nginx/fastcgi.conf;
}
```

Site file `/usr/local/etc/nginx/sites-available/default`:

```conf
server {
  listen 80;
  server_name example.local;
  root /var/www/pet/public;
  access_log /usr/local/var/logs/nginx/default.access.log main;
  index index.php index.html index.htm;
  location / { try_files $uri $uri/ /index.php?$query_string; }
  location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
  }
}
```

Enable the site by symlinking into `sites-enabled`, configure php-fpm and xdebug in `php.ini`, and set up VS Code’s PHP debug extension.

