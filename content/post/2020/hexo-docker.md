
---
layout: post
title: 手把手教你用Docker搭建Hexo博客
category: docker
date: 2018-05-18
tags:

- hexo
- docker

---

# 手把手教你用Docker搭建Hexo博客

## hexo

- 快速、简洁且高效的博客框架,Node.js 所带来的超快生成速度，让上百个页面在几秒内瞬间完成渲染。

- 只需一条指令即可部署到 GitHub Pages, Heroku 或其他网站。

- Hexo 拥有强大的插件系统，安装插件可以让 Hexo 支持 Jade, CoffeeScript。

```sh
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```

以上来自[Hexo官网](https://hexo.io/zh-cn/)

到这里,你的hexo博客已经初始化好了, blog/public文件夹下面已经生成了对应的HTML文件.

扩展阅读:

- [hexo next主题](http://theme-next.iissnan.com/)

- [hexo next主题配置](http://theme-next.iissnan.com/theme-settings.html)

## docker 部署

不BB这么多,先上Dockerfile

```docker
# node环境镜像
FROM node:latest AS build-env
# 创建hexo-blog文件夹且设置成工作文件夹
RUN mkdir -p /usr/src/hexo-blog
WORKDIR /usr/src/hexo-blog
# 复制当前文件夹下面的所有文件到hexo-blog中
COPY . .
# 安装 hexo-cli
RUN npm --registry=https://registry.npm.taobao.org install hexo-cli -g && npm install
# 生成静态文件
RUN hexo clean && hexo g

# 配置nginx
FROM nginx:latest
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
WORKDIR /usr/share/nginx/html
# 把上一部生成的HTML文件复制到Nginx中
COPY --from=build-env /usr/src/hexo-blog/public /usr/share/nginx/html
EXPOSE 80

```

接着跑一下看看.

```sh
docker build -t 镜像名:latest .;
docker run -p 80:80 -d 镜像名:latest;

```

好了,完事....

## Nginx https证书配置

最后Nginx配置https证书的步骤.

首先,你要有个证书,哪来的我不管了.

PS:良心推荐[https://freessl.org/](https://freessl.org/)直接生成免费证书

然后nginx.conf如下:

```config

events {
    worker_connections 1024;
}

http {
    server {
            listen 443;
            server_name  codelover.link;
            root /usr/share/nginx/html;
            index index.html index.htm;
            ssl on;
            ssl_certificate /etc/nginx/full_chain.pem;
            ssl_certificate_key /etc/nginx/private.key;


            include       /etc/nginx/mime.types;
            default_type  application/octet-stream;
            gzip on;
            gzip_min_length 5k;
            gzip_buffers 4 16k;
            gzip_http_version 1.1;
            gzip_comp_level 3;
            gzip_types text/plain application/json application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
            gzip_vary on;
    }
    server {
        listen 80;
        server_name  codelover.link;
        root /usr/share/nginx/html;
        index index.html index.htm;
        include  /etc/nginx/mime.types;
        default_type  application/octet-stream;
    }

}
```

这时候用docker跑你的hexo-blog镜像的时候把对应的pem和key文件映射到对应路径记录.

如下:

```sh
# codelover-blog 为配置文件路径,codelover-blog/ssl为证书路径
docker run -p 80:80 -p 443:443 \
--name codelover-blog \
-v ~/docker-data/codelover-blog/nginx.conf:/etc/nginx/nginx.conf \
-v ~/docker-data/codelover-blog/ssl/full_chain.pem:/etc/nginx/full_chain.pem \
-v ~/docker-data/codelover-blog/ssl/private.key:/etc/nginx/private.key \
--restart=always -d 你的hexo-blog博客镜像;
```

顺手也把非静态文件的nginx配置放一份,如下:

```conf
events {
    worker_connections 1024;
}

http {
    upstream webservers {
        #weigth参数表示权值，权值越高被分配到的几率越大
        #本机上的Squid开启3128端口
        server 10.31.160.197:8080 weight=5;
        server 192.168.0.1:9090  weight=3;
    }


    server {
            listen 443;
            server_name  woyaozufang.live;
            location / {
                proxy_pass   http://webservers;
            }
            ssl on;
            ssl_certificate /etc/nginx/full_chain.pem;
            ssl_certificate_key /etc/nginx/private.key;
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
            root   html;
            }
            gzip on;
            gzip_min_length 5k;
            gzip_buffers 4 16k;
            gzip_http_version 1.1;
            gzip_comp_level 3;
            gzip_types text/plain application/json application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
            gzip_vary on;
    }


    server {
        listen 8080;
        server_name  woyaozufang.live;
        location / {
            proxy_pass   http://webservers;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    server {
        listen 80;
        server_name  woyaozufang.live;
        location / {
            proxy_pass   http://webservers;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

}

```