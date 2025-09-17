---
title: "Start Your Android Tinkering Journey with Magisk"
description: "Android, Magisk, Root"
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

## Preface

Don't ask why to do this. If you're reading this article and interested in continuing, then you have this need too.

## What is Magisk

### Introduction

Magisk is a suite of open source software for customizing Android, supporting devices higher than Android 6.0.

Some highlight features:

- MagiskSU: Provide root access for applications
- Magisk Modules: Modify read-only partitions by installing modules
- MagiskBoot: The most complete tool for unpacking and repacking Android boot images
- Zygisk: Run code in every Android applications' processes

Powered by [Magisk](https://github.com/topjohnwu/Magisk)

## Operation Steps Preview

- Unlock BL
- Install Magisk App
- Download corresponding system version ROM package and extract boot.img
- Use Magisk to modify boot.img
- Flash the modified boot.img
- Install Magisk modules
- Enjoy the "free new world"

## Unlock BL

- Each has its own way, handle it yourself.
- Xiaomi devices can refer to [Xiaomi Unlock BL](https://www.miui.com/unlock/)
- OnePlus devices can refer to [OnePlus Amu](https://www.oneplus.com/cn/developer)
- Other devices please search yourself

In 2024, the best device to unlock is OnePlus. Others are inferior.

(Originally considered buying Xiaomi 14, but BL lock deterred a 10-year Xiaomi fan)

## Install Magisk App

- Download [Magisk/releases](https://github.com/topjohnwu/Magisk/releases)

After downloading, install on your phone.

```sh
adb install Magisk-v26.4.apk
```

## Extract boot.img

- Download corresponding system version ROM package and extract boot.img

If it's a flash package, directly unzip and find boot.img

If it's a line brush package, unzip to get payload.bin, need to use tools to extract.

On Windows, use FastbootEnhance tool; on Mac/Linux, use payload_dumper tool.

Basically similar.

## Use Magisk to Generate Patched boot.img

- Magisk App -> Install -> Install -> Select and Patch a File -> Select boot.img
- Wait for patched boot.img to be generated, automatically saved to /sdcard/Download/magisk_patched-xxx.img

## Flash the Modified boot.img

- Phone enters Fastboot mode, connect to computer

```sh
# Check if device is connected successfully
fastboot devices

# Flash the modified boot.img
fastboot flash boot /sdcard/Download/magisk_patched-xxx.img
```

After flashing, reboot phone.

## Activate Magisk

After rebooting, reopen Magisk App, first time should prompt to start repair.

Normally, after repair completes, phone will reboot, and after reboot, it's done.

## Install Magisk Modules

- Download Magisk modules, install locally.

The general process is basically this.

## Common Commands List

```sh
# Enter Fastboot mode
adb reboot bootloader

# Check if fastboot device is connected successfully
fastboot devices

# Unlock
fastboot oem unlock

# Unlock BL for other devices
fastboot flashing unlock

# Flash boot.img
fastboot flash boot ./Download/magisk_patched-xxx.img

# Reboot
fastboot reboot
```

## LineageOS Build Archive Download

- [LineageOS Old Versions](https://lineage-archive.timschumi.net/)
- [LineageOS Latest Versions](https://download.lineageos.org/devices/sagit/builds)

## LineageOS Set Network Detection Address

Change captive connection verification server

```sh
adb shell settings delete global captive_portal_https_url
adb shell settings delete global captive_portal_http_url
adb shell settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204
adb shell settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204
```
