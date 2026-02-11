---
title: "How to Build Your IP Proxy Pool 2025 Edition"
description: "Steps and precautions for building an IP proxy pool"
slug: how-to-build-your-ip-proxy-en
date: 2025-09-17 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - 2025
  - ip_proxy
  - proxy IP pool
  - linux
---

When performing tasks such as web crawling or data collection, using an IP proxy pool can effectively hide your real IP and avoid being banned by target websites.

This article will introduce how to build a simple IP proxy pool.

## 1. Preparation

Before starting, ensure you have a dial-up VPS (Virtual Private Server).

The price of dial-up VPS is basically from dozens to hundreds of yuan, and the optional systems are Windows or Linux.

VPS service providers basically help you configure the dial-up environment before delivery, and you can simply use the pppoe command to redial.

## 2. Building the Proxy Server

Generally, we won't directly expose the proxy service on the VPS because it basically doesn't give you a public IP.

Therefore, we need a proxy server to relay traffic.

At the same time, we need a set of "proxy server software" to connect to the dial-up VPS internally and provide proxy services externally.

Here we use [majora3 proxy server](https://github.com/yint-tech/majora-open) to build our proxy server.

----

Majora is a complete proxy IP construction cluster solution for the proxy IP pool supply chain system.

If you have a large number of network devices that can access the internet (VPS servers, router devices, mobile phones, etc.),

Then you can use these network devices to conveniently build your proxy IP pool.

Majora can meet the following application scenarios:

- If you are a proxy IP supplier with a large number of VPS nodes, you can quickly build your proxy IP pool through Majora
- You can flash the Majora terminal program into router and other network devices, so you can build your home IP resource pool or ADSL IP resource pool through Majora
- You can install Majora's apk, or use Majora's SDK integrated into mobile programs, so you can quickly build your mobile outlet IP resource pool through Majora (if your phone is rooted, you can also have the function of timed redial)

### Majora Features

- Easy to use: All terminal nodes are one-click installation to access the proxy IP cluster, no need for complex parameter configuration on the server
- Multi-supplier and purchaser concept: You can let multiple suppliers holding network devices access the Majora system at the same time, and also let multiple IP users access Majora's IP resources. Majora records the detailed billing statements of each supplier and consumer, and you can settle fees with suppliers and purchasers based on the statements
- Complete protocol: The system fully supports http/https/socks5 proxy protocols, and all proxy ports automatically identify the proxy protocol. No need for users to manually select ports
- IP quality reliability: Majora exposes port ranges and provides services in the form of tunnel proxies. And provides memory-level IP resource failure routing function. Except for very few cases (links interrupted in the request), Majora guarantees that each proxy port on the supply side can provide equivalent stable proxy services. There will never be situations where the proxy cannot be connected or connected but cannot be used
- Multi-network device support: Supports servers (VPS), mobile phones, routers, PCs (ordinary Windows or Mac computers)
- High concurrency and high bandwidth: NIO framework naturally supports very high throughput, and Majora's predecessor system (Rogue) has undergone single-node 100M bandwidth pressure

## majora3 server installation

### Docker installation method

```bash
# Install docker
yum install -y docker

# Download image:
docker pull registry.cn-beijing.aliyuncs.com/iinti/majora3:all-in-one
# Start majora server
docker run -d --network host -v ~/majora3-mysql-data:/var/lib/mysql registry.cn-beijing.aliyuncs.com/iinti/majora3:all-in-one

## The all-in-one image contains majora3 server and mysql database, data is saved in the host's ~/majora3-mysql-data directory
```

### Test if deployment is successful

```bash
curl http://127.0.0.1:6879/
```

Visit the website: http://your IP:Web service port such as [http://127.0.0.1:6879/](http://127.0.0.1:6879/),

For the first time opening the website, please register an account, the first registered account will become the administrator,

After registration, remember to set the "proxy authentication account password", such as admin/admin

If you can successfully open the website and complete account registration, it means the deployment is complete

### Production environment deployment

Production environment data generally uses cloud RDS and other databases to avoid single point of failure.

Majora is developed using Java and uses MySQL as the database,

If you want to manually deploy Majora, you need to manually complete the installation of Java and MySQL, and then perform related configuration

#### Installation preparation

- [Download installation package](https://github.com/yint-tech/majora-open/releases)
- Install jdk17
- Install MySQL, or purchase MySQL service
- Configuration and initialization
- Unzip the installation package

Database configuration initialization configuration in: assets/ddl.sql, please build the table initialization according to this SQL file

Conf folder related configuration

- The project uses springboot, and the project optional configuration is in conf/application.properties, please configure your database connection information here (the database is the MySQL installation and database configuration you completed in the previous step)
- conf/static/* is the front-end resources, if you want to replace the front-end web skin, you can replace the content here, majora front-end is open source, supports secondary development
- conf/static/majora-doc/* is the document resources, if you want to modify the document content, you can edit here

#### Run

Execute bin/startup.sh (if it's Windows, execute xxx.bat)

Observe if the log is normal

### 3. majora client access VPS

After logging into the majora admin backend, select "Terminal Download" in the right menu, and download the corresponding version terminal.

Currently supported versions include:

- Windows version (windows_amd64_v3)
- Linux version (linux_amd64_v3)
- Android version (android_all)
- Router version (linux_mipsle_softfloat)

Choose to download according to different chip architecture versions.

Below we take the Linux version as an example.

Download [majora3_linux_amd64_v3.tar.gz](https://majora3.iinti.cn/client/go/majora3_linux_amd64_v3.tar.gz) to local, unzip.

- Modify the tunnel_addr address in the majora.toml configuration file to your majora3 server address,

- Modify client_id to your terminal unique ID (can be seen in the "Terminal Management" in the majora admin backend)

```toml
group = "majora3"
extra = "zone_de"
client_id = "helloworld"
tunnel_addr = "majora3.iinti.cn:6879"
#tunnel_addr = "majora3.iinti.cn:6879"
# trace/debug/info/error/fatal/disabled
log_level = "info"

[redial]
# Execute script shell bash/ash/cmd
command = "bash"
# Script full path
exec_path = "/root/majora3/ppp.sh"
```

Repackage into majora3_linux_amd64_v3.tar.gz, then upload to your VPS.

Log in to the VPS directly, unzip and execute systemd.sh, automatically install as systemd service

```bash
tar zxvf majora3_linux_amd64_v3.tar.gz
cd majora3_linux_amd64_v3
./systemd.sh
```

If there is no problem, the majora client will connect to the majora server, and you can see your terminal in "Terminal Management".

At the same time, the program will automatically take over the device pppoe dial-up, timed redial.

### 4. Using Proxy IP

After completing the above steps, your proxy IP pool is built.

At this time, you need to create an IP pool in majora admin and add VPS devices to the pool.

IP Pool List

-> Create IP Pool

-> Select "IP Pool Type" + Set Port

-> Device List

-> Add Device

-> Save

After creation, you can see the IP pool you created in the "IP Pool List".

Then use the set port for testing.

```bash
curl -x majora:majora3@majora3.iinti.cn:6879 https://www.baidu.com/

curl -x majora:majora3@majora3.iinti.cn:6879 https://myip.ipip.net
```
