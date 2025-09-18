---
title: Linux du 命令
description: 如何使用 Linux du 命令获取当前文件夹中每个一级子目录的大小？
slug: linux-du
date: 2024-08-07 08:00:00+0000
categories:
  - linux
tags:
  - linux
  - 2024
  - TC
---

在 Linux 或 macOS 系统中，

你可以使用 `du` 命令来获取当前文件夹中每个一级子目录的大小。可以使用以下命令：

```bash
du -h --max-depth=1
```

- `-h` 选项表示以人类可读的格式显示大小（例如 KB、MB 或 GB）。
- `--max-depth=1` 表示只查看当前目录下的一级子目录的大小。

如果你希望仅查看当前目录的总大小，可以使用：

```bash
du -sh .
```

- `-s` 选项表示显示总计而不是逐个列出每个目录。

上述命令可以帮助你快速了解当前文件夹中每个子目录的大小。