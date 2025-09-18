---
title: macOS无损迁移PC微信聊天记录
description: macOS, 微信, 聊天记录, 迁移
slug: wechat-backup
date: 2025-06-24 23:36:00+0000
categories:
  - 人生删除指南
tags:
  - 2025
  - 微信
  - 聊天记录
  - macOS
---



手机端迁移聊天记录基本用微信内部工具完事了，
macOS跨机器迁移倒是需要折腾一下了。
研究了一下，其实只需要备份
"$HOME/Library/Containers/com.tencent.xinWeChat"

整个文件夹到新机器，
两边保持微信版本一致，
基本就没啥需要处理的。

于是在GPT的帮助下，
写了下面的一个脚本 backup_wechat.sh

```shell
#!/bin/bash
set -euo pipefail
BACKUP_SRC="$HOME/Library/Containers/com.tencent.xinWeChat"
BACKUP_DIR="/Volumes/workspace/weixin"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DEST="$BACKUP_DIR/wechat_backup_${TIMESTAMP}.tar.gz"
echo "🚀 开始备份微信聊天记录..."
# 检查微信进程
if pgrep -f WeChat > /dev/null; then
  echo "⚠️ 检测到微信正在运行，请先退出微信（Cmd + Q），然后重新运行脚本。"
  exit 1
fi
# 检查数据目录
if [ ! -d "$BACKUP_SRC" ]; then
  echo "❌ 找不到微信数据目录：$BACKUP_SRC"
  exit 1
fi
# 检查备份目录
if [ ! -d "$BACKUP_DIR" ]; then
  echo "❌ 备份目录不存在：$BACKUP_DIR"
  exit 1
fi
echo "📦 开始打包...（开始时间：$(date '+%F %T')）"
START_TIME=$(date +%s)
tar --exclude='.com.apple.containermanagerd.metadata.plist' --preserve-permissions -czf "$BACKUP_DEST" -C "$HOME/Library/Containers" com.tencent.xinWeChat
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "✅ 备份完成！"
echo "📁 文件位置：$BACKUP_DEST"
echo "⏱️ 耗时：${DURATION} 秒（$(printf '%02d:%02d' $((DURATION/60)) $((DURATION%60)))）"

```

备注：/Volumes/workspace/weixin 是挂载到本地的NAS 盘，
毕竟我电脑没有170G的可用空间了。
建议有SSD移动硬盘那就更好的，
不过我手上确实没这玩意~

还原脚本 restore_wechat.sh

```shell
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/Volumes/workspace/weixin"
BACKUP_FILE=$(ls -t "$BACKUP_DIR"/wechat_backup_*.tar.gz 2>/dev/null | head -1 || true)
TARGET_DIR="$HOME/Library/Containers/com.tencent.xinWeChat"
if [ -z "$BACKUP_FILE" ]; then
  echo "❌ 未找到备份文件，请确认 $BACKUP_DIR 目录下有微信备份文件。"
  exit 1
fi
echo "🚀 开始还原微信聊天记录..."
if pgrep -f WeChat > /dev/null; then
  echo "⚠️ 检测到微信正在运行，请先退出微信（Cmd + Q），然后重新运行脚本。"
  exit 1
fi
echo "📦 还原的备份文件：$BACKUP_FILE"
echo "📦 文件大小：$(du -sh "$BACKUP_FILE" | awk '{print $1}')"
echo "📂 开始解压并覆盖旧数据...（开始时间：$(date '+%F %T')）"
START_TIME=$(date +%s)
# 解压，覆盖旧内容（不删除原目录）
sudo tar -xzf "$BACKUP_FILE" -C "$HOME/Library/Containers"
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "✅ 还原完成！"
echo "⏱️ 耗时：${DURATION} 秒（$(printf '%02d:%02d' $((DURATION/60)) $((DURATION%60)))）"
echo "请打开微信检查聊天记录是否完整。"

``` 

整个过程甚至比手机聊天记录备份还原还要快多了。
(毕竟跑满带宽的NAS传输速度还是很快的)

微信真的....

道理来说 Windows电脑也差不多是类似的，

不过我手上基本没在Windows上折腾了，
自己看着来了~



