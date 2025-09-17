---
title: Android Boot Image Extraction Memo
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

## Extract boot.img from ROM package

If you have a ROM package, directly extract the `boot.img` file from the ROM package's payload.bin.

Generally use FastbootEnhance, graphical interface click click and done.

Many times we may not have the ROM package, or the ROM package is several GB, troublesome to download.

At this time, you can try using dsu-sideloader + system-squeak-arm64-ab-vanilla.img

Enter sideload system mode, use adb shell to extract boot.img.

- [DSU-Sideloader](https://github.com/VegaBobo/DSU-Sideloader)

dsu-sideloader is a tool or service component in Android system for **Dynamic System Updates (DSU)** mechanism, it allows users to temporarily load and run an alternative system image without affecting the current system. This is an official mechanism for developers to debug or test GSI (Generic System Image).

🔍 What is DSU (Dynamic System Updates)?

DSU is a feature introduced by Google from Android 10, allowing loading a system image (usually GSI) into dynamic partition for testing on supported devices, without destroying the current device's actual system (userdata/system partitions).

## Extract boot.img from running Android system

After installing dsu + system-squeak-arm64-ab-vanilla.img, enter sideload system mode.

In the sideload system, use the following script to extract `boot.img`:

```shell
#!/bin/bash

set -e

# Get current slot
slot=$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')

if [[ -z "$slot" ]]; then
  echo "❌ Unable to get slot_suffix"
  exit 1
fi

echo "🔍 Current boot slot: $slot"

# Get boot partition real path (use adb shell su to enter root then ls -l)
boot_dev=$(adb shell <<EOF
su
ls -l /dev/block/by-name/boot$slot | awk '{print \$NF}'
exit
EOF
)

boot_dev=$(echo "$boot_dev" | tr -d '\r' | tail -n 1)

if [[ -z "$boot_dev" ]]; then
  echo "❌ Unable to resolve boot$slot partition path"
  exit 1
fi

echo "📍 Resolved boot$slot real path: $boot_dev"

# Execute dd to extract boot.img
adb shell <<EOF
su
dd if=$boot_dev of=/sdcard/boot.img
exit
EOF

# Pull to local
echo "📥 Pulling boot.img to local..."
adb pull /sdcard/boot.img

echo "✅ Extraction completed! boot.img saved to current directory."
```
