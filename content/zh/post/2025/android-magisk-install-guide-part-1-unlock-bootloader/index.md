---
title: Magisk 面具安装手把手教程之一：解锁Bootloader + 提取boot.img
description: Magisk Installation Guide Part 1: Unlocking Bootloader
slug: android-magisk-install-guide-part-1-unlock-bootloader
date: 2025-10-20 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - magisk
  - Android
  - bootloader
  - root
---

## 前言

所有刷机的前提都是，解锁Bootloader。

在2025年的今天，国内大多数手机厂商已经不那么支持解锁Bootloader了。

不过也没有完全摁死，所以简单列举一下已知的解锁方法。

### 一加手机 OnePlus

一加手机的Bootloader解锁相对简单，开发者模式下，

直接打开“允许OEM解锁”选项，然后使用Fastboot命令解锁即可。

```shell

fastboot oem unlock

```

短期内，还没看到官方政策变化，所以一加手机依然是刷机爱好者的首选。


### realme 系列

realme 的Bootloader 叫： 深度测试模式。

在realme 官方论坛，注册一下账号，搜索一下“深度测试”，

找到那个帖子下载一个Apk，安装到手机上，打开它，然后按照提示操作即可。

然后大概等待一周左右审核，“深度测试” app看到审核状态为已通过。

这个时候在App里面进入深度测试模式，自动重启手机到 fastboot unlock 页面，

按着操作解锁一下bootloader 即可。


### 小米手机 Xiaomi

PS：噩梦级别解锁。

有官方解锁工具，走账户申请解锁。

但是：

什么社区等级5级，每半年只能解锁一次，解锁等待时间长达14天以上。

（ 咸鱼有绕过账号登录的，但是绕过之后还得等14天。）

2025年的今天，建议无非必要，没必要折腾小米手机解锁了。

当然，能买到已经解锁过的二手手机，可以直接玩。


### 其他手机

oem 开启之后，理论上是可以使用 `fastboot oem unlock` 解锁的。



## 提取 boot.img

刷面具之类的前提，是要有 `boot.img` 文件。


boot.img 文件，一般有两种方式获取：

1. 从刷机包提取

2. 从运行的 Android 系统提取

### 从刷机包提取 boot.img

如果有刷机包的话，直接从刷机包playload.bin提取 `boot.img` 文件即可。

一般用FastbootEnhance完事，图形界面点点点就搞掂。

在macOS或Linux下，也可以使用 `payload_dumper_go` 工具提取。

- [payload_dumper_go](https://github.com/xishang0128/payload-dumper-go)

```shell
# https://github.com/xishang0128/payload-dumper-go/releases
./payload_dumper_go -i <刷机包路径> -o <输出目录>
```

如果是线刷包，直接解压zip包，大概也能看到boot.img 文件。


### 从运行的 Android 系统提取 boot.img

很多时候我们可能并没有刷机包，或者刷机包动则几个GB，下载起来很麻烦。

这个时候可以尝试使用 dsu-sideloader +  system-squeak-arm64-ab-vanilla.img

进入到侧载系统模式，使用adb shell 提取boot.img.


- [DSU-Sideloader](https://github.com/VegaBobo/DSU-Sideloader)

dsu-sideloader 是 Android 系统中用于**动态系统更新（DSU, Dynamic System Updates）**机制的一个工具或服务组件，它允许用户在不影响当前系统的前提下，临时加载并运行一个替代性的系统镜像。这是开发者调试或测试 GSI（通用系统映像）的一种官方机制。


- [system-squeak-arm32_binder64-ab-vanilla.img.xz](https://github.com/TrebleDroid/treble_experimentations/releases)

Scripts to automatically build/CI/Release TrebleDroid GSI


安装好上的dsu + system-squeak-arm64-ab-vanilla.img 后，进入到侧载系统模式。

在shell 中执行，进入system安装环境。

```shell
# 
```


提取boot.img 脚本如下：


```shell
#!/bin/bash

set -e

# 获取当前 slot
slot=$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')

if [[ -z "$slot" ]]; then
  echo "❌ 无法获取 slot_suffix"
  exit 1
fi

echo "🔍 当前启动 slot: $slot"

# 获取 boot 分区真实路径（使用 adb shell su 进入 root 再 ls -l）
boot_dev=$(adb shell <<EOF
su
ls -l /dev/block/by-name/boot$slot | awk '{print \$NF}'
exit
EOF
)

boot_dev=$(echo "$boot_dev" | tr -d '\r' | tail -n 1)

if [[ -z "$boot_dev" ]]; then
  echo "❌ 无法解析 boot$slot 分区路径"
  exit 1
fi

echo "📍 解析到 boot$slot 真实路径: $boot_dev"

# 执行 dd 提取 boot.img
adb shell <<EOF
su
dd if=$boot_dev of=/sdcard/boot.img
exit
EOF

# 拉取到本地
echo "📥 正在拉取 boot.img 到本地..."
adb pull /sdcard/boot.img

echo "✅ 提取完成！boot.img 已保存到当前目录。"

```

如果上面的命令不可用，还是直接adb shell 到侧载系统里面，手动执行上面的命令也可以。

手动找到 boot 分区路径，然后使用 dd 提取即可。

## 结语

提取到 boot.img 之后，就可以继续后续的面具安装步骤了。

祝刷机愉快！













