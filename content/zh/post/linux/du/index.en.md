---
title: Linux du Command
description: How to use du to get the size of each first-level subdirectory in the current folder
slug: linux-du
date: 2024-08-07 08:00:00+0000
categories:
  - linux
tags:
  - linux
  - 2024
  - TC
---

On Linux or macOS, use `du` to get the size of each first-level subdirectory in the current folder:

```bash
du -h --max-depth=1
```

- `-h`: human-readable units (KB, MB, GB)
- `--max-depth=1`: only list first-level subdirectories

To view only the total size of the current directory:

```bash
du -sh .
```

- `-s`: show only the total summary

These commands help you quickly understand sizes of subfolders in the current directory.

