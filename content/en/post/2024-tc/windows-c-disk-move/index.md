---
title: Windows Full-Disk Backup and Migration — A Quick Guide
description: Windows, backup, migration
slug: windows-c-disk-move
date: 2024-12-27 08:00:00+0000
categories:
  - TC
tags:
  - Windows
  - Backup
---

## Windows Full-Disk Backup and Migration — A Quick Guide

A friend recently kept complaining that the laptop was running out of disk space.

After checking the model and confirming with Xiaomi support, this model has only one M.2 slot, with no extra expansion bay.

The SSD is not soldered, though, so replacing it is straightforward.

According to the teardown guides, you can swap the drive after removing the back cover screws.

### What You Need

- 4 TB M.2 SSD
- Precision screwdriver kit
- SSD enclosure (USB)
- Software: AOMEI Partition Assistant

## Workflow

- Install the new SSD into the USB enclosure.
- Use “Partition Assistant” to clone the entire disk to the new SSD.
- Open the laptop, swap the original SSD with the new one.
- Boot up and verify the system starts and runs normally.

## Clone the Entire Disk

This part is simple: open Partition Assistant, choose Disk Clone, select source and target disks, then start cloning.

With both drives being SSDs, the cloning finished in just over an hour for me.

Reference tutorials:

- No reinstall needed — migrate OS to SSD easily: https://www.disktool.cn/content-center/migrate-os-to-ssd/transfer-os-to-ssd.html
- How to quickly migrate your system to a new drive (3 methods): https://www.disktool.cn/jiaocheng/migrate-system-drive.html

The steps are very similar across guides, so I won’t repeat them.

After migration, open File Explorer and spot-check that the data looks right.

## Swap the Drive

This is basically unscrew, remove, replace, and screw back.

I sent my friend a teardown video; they handled it by themselves.

Yes — the entire process was DIY on their side, I only provided remote guidance.

## Issue #1: “No bootable device”

After the swap, the laptop showed “No bootable device”.

First thought: maybe the drive wasn’t seated properly?

We rechecked — the drive was firmly connected.

Swapped the old SSD back in — boots fine.

Swapped the new SSD in again — still “No bootable device”.

They didn’t have a PE USB handy at night, so we paused and made one the next day.

Thinking about it in the morning — likely a bootloader issue: the new disk doesn’t have valid boot records.

With a WinPE USB prepared, we booted into PE and confirmed the new SSD was detected. Capacity and partitions matched the old drive.

We first tried the built‑in boot repair tool in PE.

Rebooted — same error.

## Issue #2: MBR/boot records not right?

Back into PE, we used DiskGenius to manually repair the MBR/boot records.

Rebooted — still the same error.

Strange. What else could be missing?

Then it hit me: after disk migration, did we forget to convert the disk to GPT?

Back into PE, used DiskGenius to convert the new SSD to GPT.

Reboot — system starts normally.

## Takeaways

- Always have a WinPE toolkit ready when doing system migrations.
- After replacing the drive, pay attention to the disk partition style (GPT vs MBR).

## References

- How to repair disk boot with Laomaotao WinPE (Chinese): https://www.laomaotao.net/winpe/2020/0313/8129.html
- MBR (Master Boot Record) intro and repair (Chinese): https://zhuanlan.zhihu.com/p/419605886

