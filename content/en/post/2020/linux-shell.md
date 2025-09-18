---
layout: post
title: Linux Ops Cheatsheet — Install, Config, and Quick Commands
category: linux
date: 2017-03-25
tags:
- linux
- Shell
---

# Linux Ops Cheatsheet — Install, Config, and Quick Commands

## Ports & Processes

```sh
# Check what’s using a port
netstat -anp | grep "5000"
# Kill a process
kill -9 2553
# Run in background
nohup command &
```

## Install Shadowsocks (SS)

```sh
apt-get update
# Install ss
apt-get install python-pip 
pip install shadowsocks
# Run ss server
nohup ssserver -s <IP_ADDRESS> -k <PASSWORD> &
```

## Install MySQL 5.7

```sh
# Ref: http://tecadmin.net/install-mysql-5-on-ubuntu/

sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ondrej/mysql-5.7
sudo apt-get update
sudo apt-get install mysql-server

# Edit my.cnf to allow remote login
# /etc/mysql/my.cnf
# set
bind-address = 0.0.0.0

# Restart MySQL
/etc/init.d/mysql restart
```

## MySQL: Add Users

```sql
-- Grant all to root from anywhere
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root';
FLUSH PRIVILEGES;

-- Example: specific privileges for a host
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP ON <db>.* TO 'joe'@'10.163.225.87' IDENTIFIED BY '123';

-- Another example
GRANT ALL ON xxxDB.* TO 'xxx'@'%';
```

## MySQL error: “Checking for tables which need an upgrade, are corrupt or were not closed cleanly”

Steps:

```sh
sudo service mysql stop 
sudo /etc/init.d/apparmor reload
sudo service mysql start

# Or: if multiple mysql instances/processes are running
ps -A | grep mysql
sudo pkill mysql
ps -A | grep mysqld
sudo pkill mysqld

service mysql restart
mysql -u root -p
```

## 7z Quick Ops

```sh
# Compress
7z a -t7z -r manager.7z /home/manager/*
# Extract
7z x xx.zip
```

## Tencent Cloud: change password, allow root login

```sh
sudo passwd root
sudo vi /etc/ssh/sshd_config
sudo service ssh restart
```

