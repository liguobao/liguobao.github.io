---
title: 用 Magisk 开始你的安卓折腾之路
description: Android, Magisk, Root
slug: android-magisk-install
date: 2024-07-05 08:00:00+0000
image: logo.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
  - Android
---

## 前言

也不要问为什么要搞这个了，

你都看到这个文章的话，

还有兴趣看下去的话，

那就是你也有这个需求了。

## 什么是 Magisk

### Introduction

Magisk is a suite of open source software for customizing Android, 

supporting devices higher than Android 6.0.

Some highlight features:

- MagiskSU: Provide root access for applications
- Magisk Modules: Modify read-only partitions by installing modules
- MagiskBoot: The most complete tool for unpacking and repacking Android boot images
- Zygisk: Run code in every Android applications' processes

Powered by [Magisk](https://github.com/topjohnwu/Magisk)

## 操作步骤预览

- 解BL锁
- 安装 Magisk App
- 下载对应系统版本的ROM包提取boot.img
- 使用 Magisk 修改boot.img
- 刷入修改后的boot.img
- 安装 Magisk 模块
- 享受”自由新世界“

## 解BL锁

- 蛇有蛇路，shu有shu路，自己看着办了。
- 小米设备可以参考 [Xiaomi 解BL锁](https://www.miui.com/unlock/)
- 一加设备可以参考 [OnePlus 阿木大侠](https://www.oneplus.com/cn/developer)
- 其他设备请自行搜索

2024年，最好解锁的设备，是 OnePlus 了。

其他的，都是弟弟。

（ 本来考虑要不买台小米14的，因为BL锁劝退了十年米粉）



## 安装 Magisk App

- 下载 [Magisk/releases](https://github.com/topjohnwu/Magisk/releases)

下载后，自行安装到手机上。

```sh 

adb install Magisk-v26.4.apk
```

## 提取boot.img

- 下载对应系统版本的ROM包提取boot.img

如果是卡刷包，可以直接解压，找到boot.img

如果是线刷包，解压之后得到的是 playload.bin，需要使用工具解压。

Windows上可以使用FastbootEnhance工具，Mac/Linux上可以使用payload_dumper工具。

基本大同小异。

```sh

## 使用 Magisk 生成修补后的boot.img

- Magisk App -> Install -> Install -> Select and Patch a File -> 选择boot.img
- 等待生成修补后的boot.img，自动保存到 /sdcard/Download/magisk_patched-xxx.img

## 刷入修改后的boot.img

- 手机进入Fastboot模式，连接电脑

```sh
# 检查设备是否连接成功
fastboot devices;

# 刷入修改后的boot.img
fastboot flash boot /sdcard/Download/magisk_patched-xxx.img

```

刷入完成后，重启手机。

## Magisk 激活

重启手机之后，重新打开Magisk App，第一次应该会提示需要启动修复。

正常情况下，修复完成后会重启手机，再次开机后就完事了。

## 安装 Magisk 模块

- 下载面具模块，在本地安装即可。

大体流程基本如此。


## 常用命令列表

```sh
# 进入Fastboot模式
adb reboot bootloader
# 检查fastboot设备是否连接成功
fastboot devices

#解锁
fastboot oem unlock

# 其他设备解BL锁
fastboot flashing unlock

# 刷入boot.img
fastboot flash boot /sdcard/Download/magisk_patched-xxx.img

# 重启
fastboot reboot

```
