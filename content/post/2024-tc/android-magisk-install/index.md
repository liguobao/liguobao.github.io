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