---
title: 如何搭建你的IP代理池2025版
description: 搭建IP代理池的步骤和注意事项
slug: how-to-build-your-ip-proxy
date: 2025-09-17 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - 2025
  - ip_proxy
  - 代理IP池
  - linux
---

在进行网络爬虫、数据采集等任务时，使用IP代理池可以有效地隐藏真实IP，避免被目标网站封禁。

本文将介绍如何搭建一个简单的IP代理池。

## 1. 准备工作

在开始之前，确保你有一台拨号VPS（虚拟专用服务器）。

拨号VPS的价格基本是几十块到百来块不等，可选的系统是Windows Or Linux。

VPS 服务商交付之前基本会帮你配置好拨号环境，简单使用pppoe 命令即可重播。

## 2. 搭建代理服务器

一般不会直接在VPS上暴露代理服务，因为这玩意基本是不给你公网IP的。

于是，我们需要一个代理服务器来中转流量。

与此同时，需要一套“代理服务器软件”对内连接拨号VPS，对外提供代理服务。

这里我们使用的 [majora3 proxy server](https://github.com/yint-tech/majora-open) 用来搭建我们的代理服务器。

----

Majora 是一套完整的代理 ip 建设集群方案，为代理 IP 池供应链系统，

如果您拥有大批量的可以上网的网络设备（VPS 服务器、路由器设备、手机等移动设备），

那么你可以使用这些网络设备方便的构建您的代理 IP 池。

Majora 可以满足的应用场景如下：

- 您是代理 ip 供应商，拥有大量 VPS 节点，则你可以通过 Majora 快速搭建您的代理 IP 池
- 您可以将 Majora 终端程序刷入到路由器等网络设备中，则您可以通过 Majora 构建您的家庭 ip 资源池或者 ADSL IP 资源池
- 您可以安装 Majora 的 apk，或者使用 Majora 的 sdk 集成到手机程序中，则您可以通过 Majora 快速搭建您的手机出口 IP 资源池(如果您的手机有 root，那么还可以拥有定时重播的功能)

### Majora 功能特点

- 使用简单：所有终端节点都是一键安装即可接入代理 ip 集群，无需服务器复杂的参数配置
- 多供应商和采购商概念：您可以让多个持有网络设备的供应商同时接入 Majora 系统，同时可以让多个 ip 使用方接入 Majora 的 IP 资源。Majora 记录了每个供应者和消费者的详细账单流水，您可以根据流水对供应商和采购商进行费用结算
- 协议完整：本系统完整支持 http/https/socks5 代理协议，所有代理端口均自动识别代理协议。无需用户手动选择端口
- ip 质量可靠性：Majora 暴露端口范围，以隧道代理的方式提供服务。并提供内存级别的 IP 资源失败路由功能。除极少情况（被中断请求中的链接），Majora 保证供应端每个代理端口均可提供对等的稳定代理服务。永远不会出现代理链接不上，或者连接上无法使用的情况
- 多网络设备支持：支持服务器（VPS)、手机、路由器、PC(普通 windows 或者 mac 电脑)
- 高并发和高带宽：NIO 框架天然支持非常高的吞吐，且 Majora 的前序系统(Rogue)已经经过单节点百M 的带宽压力

## majora3 server 安装

### docker 方式安装

```bash

# 安装docker
yum install -y docker

# 下载镜像:
docker pull registry.cn-beijing.aliyuncs.com/iinti/majora3:all-in-one
# 启动majora服务器
docker run -d --network host -v ~/majora3-mysql-data:/var/lib/mysql registry.cn-beijing.aliyuncs.com/iinti/majora3:all-in-one

## all-in-one镜像包含了majora3 server和mysql数据库，数据保存在宿主机的~/majora3-mysql-data目录下

```

### 测试是否部署成功

```bash
curl http://127.0.0.1:6879/
```

访问网站： http://你的IP:Web服务端口 如(http://127.0.0.1:6879/)[http://127.0.0.1:6879/],

首次打开网站请注册账户，第一个注册账户将会成为管理员,

注册完成后记得设置”代理鉴权账号密码“, 如 admin/admin

若成功打开网站，完成账户注册，则代表部署完成

### 生产环境部署

生产环境数据一般使用云端RDS等数据库，避免单点故障。

Majora 使用 java 开发，并且使用 mysql 作为数据库, 

如果你想手动部署 Majora，则需要手动完全 java 和 mysql 的安装，之后进行相关配置

#### 安装准备

- [下载安装包](https://github.com/yint-tech/majora-open/releases)
- 安装 jdk17
- 安装 mysql，或者购买 mysql 服务
- 配置和初始化
- 解压安装包

数据库配置初始化配置在:assets/ddl.sql,请根据本 sql 文件进行数据库建表初始化

conf 文件夹的相关配置

- 项目使用 springboot，其中项目可选配置在 conf/application.properties，请在这里配置您的数据库链接信息（数据库为您上一步完成的 mysql 安装和数据库配置）
- conf/static/*为前端资源，如果你想替换前端网页皮肤，则可以替换这里的内容 majora 前端是开源的，支持二开的
- conf/static/majora-doc/*为文档资源，如果你想修改文档内容，则可以编辑这里

#### 运行

执行bin/startup.sh (如果是 windows，那么执行 xxx.bat 即可)

观察日志是否正常

### 3. majora 客户端接入VPS

登录majora admin 后台后，在右侧菜单中选择“终端下载”，下载对应版本终端。

目前支持的版本有：

- Windows 版（windows_amd64_v3）
- Linux 版（linux_amd64_v3）
- Android 版（android_all）
- 路由器版（linux_mipsle_softfloat）

根据不同的芯片架构版本选择下载即可。

以下我们以Linux版本为例。

下载[majora3_linux_amd64_v3.tar.gz](https://majora3.iinti.cn/client/go/majora3_linux_amd64_v3.tar.gz)到本地，解压。

- 修改 majora.toml 配置文件中的 tunnel_addr 地址为你的 majora3 server 地址，

- 修改 client_id 为你的终端唯一ID（在 majora admin 后台的“终端管理”中可以看到）

```toml
group = "majora3"
extra = "zone_de"
client_id = "helloworld"
tunnel_addr = "majora3.iinti.cn:6879"
#tunnel_addr = "majora3.iinti.cn:6879"
# trace/debug/info/error/fatal/disabled
log_level = "info"

[redial]
# 执行脚本的shell bash/ash/cmd
command = "bash"
# 脚本完整路径
exec_path = "/root/majora3/ppp.sh"
```

重打包成majora3_linux_amd64_v3.tar.gz, 然后上传到你的VPS上。

直接登录到VPS测，解压后执行 systemd.sh, 自动安装成 systemd 服务

```bash
tar zxvf majora3_linux_amd64_v3.tar.gz
cd majora3_linux_amd64_v3
./systemd.sh
```

没问题的情况下，majora client 会连接到 majora server, 并且在“终端管理”中可以看到你的终端。

同时，程序会自动接管设备pppoe拨号，定时重拨。

### 4. 使用代理IP

完成以上步骤后，你的代理IP池就搭建完成了。

这个时候需要在majora admin 中建立IP池，并将VPS设备添加到池中。

IP池列表

-> 创建IP池

-> 选择“IP池类型” + 设定端口

-> 设备列表

-> 添加设备

-> 保存

创建完成后，可以在“IP池列表”中看到你创建的IP池。

然后使用设定的端口进行测试。

```bash

curl -x majora:majora3@majora3.iinti.cn:6879 https://www.baidu.com/

curl -x majora:majora3@majora3.iinti.cn:6879 https://myip.ipip.net

```