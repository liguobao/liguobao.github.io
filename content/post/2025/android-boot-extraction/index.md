---
title: Android boot 镜像提取备忘录
description: Android boot image extraction
slug: android-boot-image-extraction
date: 2025-06-24 23:36:00+0000
categories:
  - TC
tags:
  - 2025
  - android
  - boot
---

## 从刷机包提取 boot.img

如果有刷机包的话，直接从刷机包playload.bin提取 `boot.img` 文件即可。

一般用FastbootEnhance完事，图形界面点点点就搞掂。

很多时候我们可能并没有刷机包，或者刷机包动则几个GB，下载起来很麻烦。

这个时候可以尝试使用 dsu-sideloader +  system-squeak-arm64-ab-vanilla.img 

进入到侧载系统模式，使用adb shell 提取boot.img.

- [DSU-Sideloader](https://github.com/VegaBobo/DSU-Sideloader)

dsu-sideloader 是 Android 系统中用于**动态系统更新（DSU, Dynamic System Updates）**机制的一个工具或服务组件，它允许用户在不影响当前系统的前提下，临时加载并运行一个替代性的系统镜像。这是开发者调试或测试 GSI（通用系统映像）的一种官方机制。

🔍 什么是 DSU（Dynamic System Updates）？

DSU 是 Google 从 Android 10 开始引入的功能，允许在支持的设备上加载一个系统镜像（通常是 GSI）到动态分区中进行测试，而不会破坏当前设备的实际系统（userdata/system 分区）。

## 从运行的 Android 系统提取 boot.img

安装好上的dsu + system-squeak-arm64-ab-vanilla.img 后，进入到侧载系统模式。

在侧载系统中，使用以下脚本提取 `boot.img`：

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