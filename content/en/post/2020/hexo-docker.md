---
layout: post
title: Build and Deploy a Hexo Blog with Docker
category: docker
date: 2018-05-18
tags:
- hexo
- docker
---

# Hexo

Install and scaffold:

```sh
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```

Docs: https://hexo.io

## Docker Deployment

Dockerfile:

```dockerfile
FROM node:latest AS build-env
RUN mkdir -p /usr/src/hexo-blog
WORKDIR /usr/src/hexo-blog
COPY . .
RUN npm --registry=https://registry.npm.taobao.org install hexo-cli -g && npm install
RUN hexo clean && hexo g

FROM nginx:latest
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
WORKDIR /usr/share/nginx/html
COPY --from=build-env /usr/src/hexo-blog/public /usr/share/nginx/html
EXPOSE 80
```

Build and run:

```sh
docker build -t hexo-blog:latest .
docker run -p 80:80 -d hexo-blog:latest
```

## Nginx TLS Config

Obtain a cert (e.g., via https://freessl.org/), then use an nginx.conf like:

```nginx
events { worker_connections 1024; }
http {
  server {
    listen 443;
    server_name example.com;
    root /usr/share/nginx/html;
    index index.html index.htm;
    ssl on;
    ssl_certificate /etc/nginx/full_chain.pem;
    ssl_certificate_key /etc/nginx/private.key;
    include /etc/nginx/mime.types;
    gzip on; gzip_min_length 5k; gzip_buffers 4 16k; gzip_http_version 1.1;
    gzip_comp_level 3;
    gzip_types text/plain application/json application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
    gzip_vary on;
  }
  server {
    listen 80; server_name example.com;
    root /usr/share/nginx/html; index index.html index.htm;
    include /etc/nginx/mime.types;
  }
}
```

Run with mounts for config and certs:

```sh
docker run -p 80:80 -p 443:443 \
  --name hexo-site \
  -v ~/docker-data/site/nginx.conf:/etc/nginx/nginx.conf \
  -v ~/docker-data/site/ssl/full_chain.pem:/etc/nginx/full_chain.pem \
  -v ~/docker-data/site/ssl/private.key:/etc/nginx/private.key \
  --restart=always -d hexo-blog:latest
```

Example reverse-proxy config (non-static): define upstreams and proxy_pass in server blocks as needed.

