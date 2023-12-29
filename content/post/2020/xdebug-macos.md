---
layout: post
title: 用Visual Studio Code Debug世界上最好的语言(Mac篇)
category: PHP
date: 2018-05-21
tags:
- Visual Studio Code
- debug
- PHP
- xdebug
---
# 用Visual Studio Code Debug世界上最好的语言(Mac篇)

首先,你要有台Macbook Pro,接着才继续看这个教程.

PS:Windows用户看这里[用Visual Studio Code Debug世界上最好的语言](https://zhuanlan.zhihu.com/p/25844268)

## brew 环境准备

见[brew.sh](https://brew.sh/index_zh-cn),或者

```sh
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## PHP7 + nginx + php-fpm + xdebug

### PHP7

```sh

brew install php@7.1

```

安装完了之后看下安装路径:

```sh

where php;

##➜  ~ where php
##   /usr/local/opt/php@7.1/bin/php
##   /usr/bin/php

```

一般php.ini在/usr/local/etc/php/7.1

```sh
ls /usr/local/etc/php/7.1
#conf.d       pear.conf    php-fpm.conf php-fpm.d    php.ini
```

待会我们配置xdebug和php-fpm的时候会用到这个这些配置文件的,先跳过

## xdebug安装

本来其实一句brew install php71-xdebug --without-homebrew-php就完事的,谁知道homebrew-php最近被移除了,所以就尴尬了...

手动去下载xdebug然后配置吧.下载页面:[https://xdebug.org/files/](https://xdebug.org/files/)

选择自己要安装的版本,我这里选了2.6.

```sh
# 创建一个你喜欢的路径存放,我放在了~/tool/目录下;
mkdir tool;

wget https://xdebug.org/files/xdebug-2.6.0.tgz;

# 解压
tar xvzf xdebug-2.6.0.tgz;

cd xdebug-2.6.0;

# 初始化php模块
phpize;

# 生成对应的so文件
# ./configure --enable-xdebug --with-php-config=PHP安装路径/bin/php-config;
./configure --enable-xdebug --with-php-config=/usr/local/Cellar/php@7.1/7.1.17/bin/php-config;

make;

make test;

# 上一步正常make执行完毕之后会在xdebug-2.6.0/modules/文件夹下生成xdebug.la和xdebug.so,待会我们在php.ini中配置xdebug会用到这个文件

```

[https://www.techflirt.com/install-configure-xdebug-on-xampp-windows-and-mac/](https://www.techflirt.com/install-configure-xdebug-on-xampp-windows-and-mac/)

## 安装nginx

```sh

brew install nginx

```

### 配置nginx.conf

安装完成之后开始配置nginx,首先创建一堆需要用到的文件件.

```sh
mkdir -p /usr/local/var/logs/nginx
mkdir -p /usr/local/etc/nginx/sites-available
mkdir -p /usr/local/etc/nginx/sites-enabled
mkdir -p /usr/local/etc/nginx/conf.d
mkdir -p /usr/local/etc/nginx/ssl
sudo mkdir -p /var/www
sudo chown :staff /var/www
sudo chmod 777 /var/www

#作者：GQ1994
#链接：https://www.jianshu.com/p/a642ee8eca9a
#來源：简书
#著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

然后vim /usr/local/etc/nginx/nginx.conf 输入以下内容：

```conf
user root wheel; #默认的是nobody会导致403
worker_processes  1;

error_log   /usr/local/var/logs/nginx/error.log debug;


pid        /usr/local/var/run/nginx.pid;


events {
    worker_connections  256;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /usr/local/var/logs/access.log  main;

    sendfile        on;
    keepalive_timeout  65;
    port_in_redirect off;

    include /usr/local/etc/nginx/sites-enabled/*;
}
```

### 设置nginx php-fpm配置文件

vim /usr/local/etc/nginx/conf.d/php-fpm
修改为(没有则创建)

```conf
#proxy the php scripts to php-fpm
location ~ \.php$ {
    try_files                   $uri = 404;
    fastcgi_pass                127.0.0.1:9000;
    fastcgi_index               index.php;
    fastcgi_intercept_errors    on;
    include /usr/local/etc/nginx/fastcgi.conf;
}

```

### 创建默认虚拟主机default

vim /usr/local/etc/nginx/sites-available/default输入：

```conf
server {
listen       80;#如果80被用了可以换成别的,随你开心
server_name  www.qilipet.com admin.qilipet.com;
root   /var/www/pet/public;

access_log  /usr/local/var/logs/nginx/default.access.log  main;
index index.php index.html index.htm;

location / {
            # First attempt to serve request as file, then
            # as directory, then fall back to displaying a 404.
            try_files $uri $uri/ /index.php?$query_string;
            # Uncomment to enable naxsi on this location
            # include /etc/nginx/naxsi.rules
    }

location ~ \.php$ {
            fastcgi_pass  127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include    fastcgi_params;
    }
}
```

此部分内容基本来自[GQ1994:mac下配置php、nginx、mysql、redis](https://www.jianshu.com/p/a642ee8eca9a)

## 配置php.ini

回到我们的/usr/local/etc/php/7.1文件夹

在php.ini中加入xdebug配置

```ini

[xdebug]
;zend_extension="刚刚的xdebug路径/modules/xdebug.so"
zend_extension="~/tool/xdebug-2.6.0/modules/xdebug.so"
xdebug.remote_enable = 1
xdebug.remote_autostart = 1
xdebug.remote_connect_back = 1
;默认的9000已经被php-fpm占用了,切记换一个端口
xdebug.remote_port = 9001
xdebug.scream = 0
xdebug.show_local_vars = 1
```

重启一下php-fpm和nginx,看一下php是不是都正常跑起来了.

## VS Code配置

### User Settings配置PHP目录

```json
  "php.executablePath": "/usr/local/opt/php@7.1/bin/php"
```

### 安装php debug插件

安装完成之后配置一下launch.json

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [

        {
            "name": "Listen for XDebug",
            "type": "php",
            "request": "launch",
            "port": 9001 //默认是9000已经被php-fpm占用,上一步我们配置远程端口是9001
        },
        {
            "name": "Launch currently open script",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "port": 9001 //默认是9000已经被php-fpm占用,上一步我们配置远程端口是9001
        }
    ]
}
```

然后就愉快debug最好的语言吧!

![debug](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-21%20%E4%B8%8B%E5%8D%8810.16.58.png)

## 其他部分

- [macOS系统PHP7增加Xdebug](https://itony.net/post/osx-php-xdebug.html)

- [Install PEAR and PECL on Mac OS X](https://www.tuicool.com/articles/bE77N3)

- [Xdebug on macOS 10.13 with PHP 7](https://questionfocus.com/xdebug-on-macos-10-13-with-php-7.html)

- [install-configure-xdebug-on-xampp-windows-and-mac](https://www.techflirt.com/install-configure-xdebug-on-xampp-windows-and-mac/)

- [installing-pecl-and-pear-on-os-x-10-11-el-capitan-macos-10-12-sierra-macos-10](https://stackoverflow.com/questions/32893056/installing-pecl-and-pear-on-os-x-10-11-el-capitan-macos-10-12-sierra-macos-10)