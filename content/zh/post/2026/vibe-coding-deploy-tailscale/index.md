---
title: 谁不想要自己的Tailscale内网呢~
description: 本文介绍如何使用Vibe Coding在部署Tailscale，实现跨内网路由，方便远程访问和管理设备。
slug: vibe-coding-deploy-tailscale
date: 2026-02-10 08:00:00+0000
categories:
  - linux
tags:
  - tailscale
  - vpn
  - development
  - vibe-coding
---

## 前言

正经人谁没一个k8s集群呢？对的吧。

正经人一年没崩两次集群都不算事吧？

正经人谁不这里薅一台机器哪里搞一台服务器呢？

于是，开始折腾跨“局域网内网“的搭建了。

早一些时间，已经在云端手动搭建Wireguard，

倒是几个节点都跑起来了，基本也是可用的。

但是这玩意手动部署总是过于麻烦，

一个个节点配置搞错了Debug起来都想死。

前阵子也看到过Tailscale，

用他们的官方授权服务也跑起来了Demo，

感觉是能用的，只是发现不付钱只给五台设备，

好像不太合适，数量也不太够~

后来看了下运维小哥最近在折腾Tailscale 自建授权，

平时我们的机器互访也跑起来了，

突然就想懂应该怎么做了。

于是，开整~

## Tailscale：概念先行

```md

Tailscale 本质是三层东西：

客户端（必需）

tailscale CLI / GUI（macOS / Linux / Windows / iOS / Android）

负责：WireGuard 加密、打洞、直连/中继

控制面（Control Plane）

设备注册

身份认证

下发节点列表、ACL、DERP 信息

- 官方的是 Tailscale SaaS
- 自建的是 Headscale

中继面（DERP，可选）

打洞失败时转发流量

可用官方 DERP，也可以自建
```

所以, 先来一台服务器~

某云轻量服务器走起。

## 再和GPT聊聊方案

- 和GPT聊一下，确认了可行性。

![可行性](/img/post/vibe-coding-deploy-tailscale/1.jiagou.png)

```md
用 Headscale 作为 Tailscale Control Server，

关闭 Tailscale 官方 SaaS，

鉴权接入你自己的 OAuth / LDAP / Token / SSO。


## 二、关键组件说明（重要）


### Tailscale

官方客户端 + 官方 SaaS

你不用官方 SaaS

只使用客户端（tailscale up --login-server=...）


### Headscale（核心）

Tailscale 官方协议的 开源 Control Server

100% 自托管

MIT License

支持：

Namespaces / Users

ACL

PreAuthKey

OIDC（可扩展）

👉 Headscale = 你自己的 Tailscale Server
```

道理懂了，开干吧。

## Vibe Coding DevOps 开整

想了下，

直接让GPT生成一个Vibe Coding要求，

扔给Copilot 直接务器上面跑试试看。

于是有了下面这个

```md

你是一个资深 SRE / 基础设施工程师，请帮我在一台 Linux 服务器上，
从 0 到 1 完整部署一个【自建 Tailscale 控制面系统】。

================================================
【服务器与基础信息】
================================================

服务器地址：
ts.example.cn

SSH 登录方式：
ssh root@ts.example.cn

操作系统：
假设为 Ubuntu 20.04 / 22.04（主流 Linux 即可）

权限：
- root 权限
- 可执行任意系统命令

域名状态（已完成）：
- ts.example.cn        → 本机公网 IP
- auth.example.cn      → 本机公网 IP
DNS 已解析完成，不需要你处理 DNS

HTTPS 要求：
- 使用 acme.sh 申请 Let’s Encrypt 证书
- 使用 Nginx 模式或 standalone 均可
- 需要配置自动续期
- 证书路径要清晰、可维护

================================================
【最终目标】
================================================

在这台服务器上完成以下系统部署，并确保长期稳定运行：

1. Headscale
   - 作为唯一的 Tailscale 控制面（替代官方 SaaS）
   - 对外服务地址：https://ts.example.cn

2. 账号 / 密码登录系统
   - 使用 Authentik
   - 通过 OIDC 与 Headscale 集成
   - 登录入口：https://auth.example.cn

3. PostgreSQL
   - 仅供 Authentik 使用

4. Tailscale Client
   - 本机作为一个普通 Tailscale 节点加入网络

5. HTTPS
   - 由宿主机 Nginx 统一负责
   - 不使用 Traefik / Caddy

6. 容器管理
   - 使用 Docker + docker-compose
   - 所有服务可一键启动 / 停止

最终效果应支持：
tailscale up --login-server=https://ts.example.cn

================================================
【强制架构约束（必须遵守）】
================================================

- ❌ 不使用官方 Tailscale SaaS
- ✅ 使用 Headscale 作为唯一控制面
- ✅ 使用 Authentik 提供账号 / 密码登录
- ✅ Headscale + Authentik + PostgreSQL 使用 docker-compose
- ✅ Tailscale client 使用官方 tailscale 容器
- ✅ HTTPS 完全由宿主机 Nginx 提供
- ❌ 不在该机器上运行 k3s master / etcd
- ✅ 所有配置必须是真实可运行配置，不允许伪代码

================================================
【域名与端口规划（固定）】
================================================

对外域名：
- Headscale： https://ts.example.cn
- 鉴权入口： https://auth.example.cn

内部端口（可自行选择，但需合理）：
- Headscale：8080
- Authentik：9000
- PostgreSQL：5432

================================================
【你需要输出的内容（一步都不能少）】
================================================

请你【严格按顺序】输出并解释以下内容：

1. 🧱 服务器初始化
   - 更新系统
   - 安装 Docker
   - 安装 docker-compose（plugin 或 standalone）
   - 安装 Nginx
   - 安装 acme.sh

2. 🔐 HTTPS 证书申请
   - 使用 acme.sh 为以下域名申请证书：
     - ts.example.cn
     - auth.example.cn
   - 给出完整、真实的命令
   - 配置自动续期
   - 明确证书存放路径

3. 🌐 Nginx 配置
   - 为 ts.example.cn 生成 server block（反代 Headscale）
   - 为 auth.example.cn 生成 server block（反代 Authentik）
   - 启用 HTTPS
   - 设置正确的 proxy headers（Host / X-Forwarded-Proto 等）

4. 📁 目录结构规划
   - 给出一个推荐的部署目录（例如 /opt/headscale-stack）
   - 输出完整目录树

5. 🐳 Docker Compose
   - PostgreSQL（仅 Authentik 使用）
   - Authentik（账号 / 密码 + OIDC）
   - Headscale
   - Tailscale client
   - 包含：
     - volumes
     - networks
     - restart policy

6. ⚙️ Headscale 配置文件
   - server_url 设置为：https://ts.example.cn
   - IP 段使用：100.64.0.0/10
   - 启用 OIDC
   - OIDC issuer 设置为：
     https://auth.example.cn/application/o/headscale/

7. 👤 Authentik 初始化与 OIDC 配置
   - 如何首次初始化管理员账号
   - 如何创建 OIDC Application
   - Client ID 应设置为：headscale
   - Redirect URI 必须是：
     https://ts.example.cn/oidc/callback
   - 如何获取 Client Secret 并填回 Headscale

8. ▶️ 启动与验证
   - docker compose up -d
   - 如何检查容器状态
   - 如何验证：
     - https://ts.example.cn 可访问
     - https://auth.example.cn 登录页正常

9. 🧪 客户端接入验证
   - 给出真实命令：
     tailscale up --login-server=https://ts.example.cn
   - 描述完整登录流程：
     CLI → 浏览器 → 账号密码 → 回调 → 入网

10. ⚠️ 关键注意事项
    - 哪些目录必须备份
    - 为什么 2C2G 资源足够
    - 哪些事情绝对不能做（例如跑 k3s master / etcd）

================================================
【输出风格要求】
================================================

- 必须是「一步一步可执行的实操手册」
- 所有命令、配置都可以直接复制执行
- 不允许说“自行配置”“略”
- 假设读者是懂 Linux / Docker / 网络的工程师
- 以工程实战为第一优先级

================================================
【最终验收标准】
================================================

当我完全照你给出的步骤执行后，应当满足：

- https://ts.example.cn 正常工作（Headscale）
- https://auth.example.cn 正常登录（账号 / 密码）
- 新机器可通过账号密码加入 Tailscale 网络
- 该网络可稳定承载 k3s over Tailscale

请严格按照以上要求输出完整部署方案。


```

云服务买好了，把上面两个域名解析到新机器，

开整...

最终的成果的核心Docker-compose是如下：

```docker-compose

version: '3.8'

services:
  postgresql:
    image: docker.io/library/postgres:16-alpine
    container_name: headscale-postgresql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:-authentik_password_change_me}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
    networks:
      - headscale-net
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: docker.io/library/redis:alpine
    container_name: headscale-redis
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - redis:/data
    networks:
      - headscale-net

  authentik-server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS:-authentik_password_change_me}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:-change_this_to_random_string_min_50_chars_abcdefghijklmnopqrstuvwxyz}
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
      AUTHENTIK_DISABLE_UPDATE_CHECK: "true"
      AUTHENTIK_DISABLE_STARTUP_ANALYTICS: "true"
      AUTHENTIK_AVATARS: "initials"
    volumes:
      - ./authentik/media:/media
      - ./authentik/custom-templates:/templates
    ports:
      - "127.0.0.1:9000:9000"
    networks:
      - headscale-net
    depends_on:
      - postgresql
      - redis

  authentik-worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS:-authentik_password_change_me}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:-change_this_to_random_string_min_50_chars_abcdefghijklmnopqrstuvwxyz}
      AUTHENTIK_ERROR_REPORTING__ENABLED: "false"
      AUTHENTIK_DISABLE_UPDATE_CHECK: "true"
      AUTHENTIK_DISABLE_STARTUP_ANALYTICS: "true"
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./authentik/media:/media
      - ./authentik/certs:/certs
      - ./authentik/custom-templates:/templates
    networks:
      - headscale-net
    depends_on:
      - postgresql
      - redis

  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    restart: unless-stopped
    command: serve
    volumes:
      - ./headscale/config.yaml:/etc/headscale/config.yaml:ro
      - ./headscale/data:/var/lib/headscale
      - ./headscale/run:/var/run/headscale
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:9090:9090"
    networks:
      - headscale-net

networks:
  headscale-net:
    driver: bridge

volumes:
  redis:
    driver: local

```

Nginx config 配置如下：

```config

upstream authentik {
    server 127.0.0.1:9000;
}

server {
    listen 80;
    server_name auth.example.cn;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name auth.example.cn;

    ssl_certificate /etc/nginx/ssl/ts.example.cn.fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/ts.example.cn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 25M;

    location / {
        proxy_pass http://authentik;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}

upstream headscale {
    server 127.0.0.1:8080;
}

map $http_upgrade $connection_upgrade {
    default keep-alive;
    'websocket' upgrade;
    '' close;
}

server {
    listen 80;
    server_name ts.example.cn;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ts.example.cn;

    ssl_certificate /etc/nginx/ssl/ts.example.cn.fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/ts.example.cn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://headscale;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $server_name;
        proxy_redirect http:// https://;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}

```

## 最终成果

```shell
*** System restart required ***
➜  ~ tailscale status
100.64.0.3   haru               https://auth.example.cn/application/o/headscale dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.9   hc1                https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.4   lgb-amd-3700       https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  idle, tx 1052 rx 860
100.64.0.2   lgb-macbookair-m4  https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  macOS  offline
100.64.0.8   rainyun   https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.10  ts-example-cn        https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.11  us-vultr           https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  idle, tx 1012 rx 860
100.64.0.6   vm-0-8-ubuntu-new  https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.5   vm-16-12-ubuntu    https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
100.64.0.7   vm-28-17-ubuntu    https://auth.example.cn/application/o/headscale/dcaba24e364c6ea7c3dd676f3d445f85843028f1f12880dcba46acabf82509ad  linux  -
➜  ~

➜  ~ ping 100.64.0.3
PING 100.64.0.3 (100.64.0.3) 56(84) bytes of data.
64 bytes from 100.64.0.3: icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from 100.64.0.3: icmp_seq=2 ttl=64 time=0.034 ms
64 bytes from 100.64.0.3: icmp_seq=3 ttl=64 time=0.043 ms
^C
--- 100.64.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2026ms
rtt min/avg/max/mdev = 0.034/0.042/0.049/0.006 ms
➜  ~ ping 100.64.0.9
PING 100.64.0.9 (100.64.0.9) 56(84) bytes of data.
64 bytes from 100.64.0.9: icmp_seq=1 ttl=64 time=737 ms
64 bytes from 100.64.0.9: icmp_seq=2 ttl=64 time=12.5 ms
64 bytes from 100.64.0.9: icmp_seq=3 ttl=64 time=12.0 ms
64 bytes from 100.64.0.9: icmp_seq=4 ttl=64 time=12.1 ms
64 bytes from 100.64.0.9: icmp_seq=5 ttl=64 time=12.9 ms
64 bytes from 100.64.0.9: icmp_seq=6 ttl=64 time=12.3 ms
64 bytes from 100.64.0.9: icmp_seq=7 ttl=64 time=12.1 ms
64 bytes from 100.64.0.9: icmp_seq=8 ttl=64 time=12.4 ms
^C
--- 100.64.0.9 ping statistics ---
8 packets transmitted, 8 received, 0% packet loss, time 7008ms
rtt min/avg/max/mdev = 12.015/102.875/736.725/239.572 ms
➜  ~ ping 100.64.0.5
PING 100.64.0.5 (100.64.0.5) 56(84) bytes of data.
64 bytes from 100.64.0.5: icmp_seq=1 ttl=64 time=96.3 ms
64 bytes from 100.64.0.5: icmp_seq=2 ttl=64 time=5.57 ms
64 bytes from 100.64.0.5: icmp_seq=3 ttl=64 time=4.93 ms
64 bytes from 100.64.0.5: icmp_seq=4 ttl=64 time=5.12 ms
64 bytes from 100.64.0.5: icmp_seq=5 ttl=64 time=5.40 ms
^C
--- 100.64.0.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 4.932/23.468/96.318/36.425 ms
➜  ~
```

## 实操注意事项

- 切记让Vibe Coding在服务器上执行操作，不然很多时候可能在当前的机器上就跑起来了
- Docker 镜像国内不太好访问，建议本地导出后导入到服务器
- authentik 创建用户可能没有自动完成，继续让Vibe Coding 生成脚本去让他操作即可

基本没什么别的坑了，

祝玩得开心~