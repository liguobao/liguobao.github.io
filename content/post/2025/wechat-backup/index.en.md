---
title: "macOS Lossless Migration of PC WeChat Chat Records"
description: "macOS, WeChat, Chat Records, Migration"
slug: wechat-backup
date: 2025-06-24 23:36:00+0000
categories:
  - Life Deletion Guide
tags:
  - 2025
  - WeChat
  - Chat Records
  - macOS
---

Migrating chat records on mobile basically uses WeChat's internal tools.

Cross-machine migration on macOS requires some fiddling.

After researching, you actually just need to backup

"$HOME/Library/Containers/com.tencent.xinWeChat"

The entire folder to the new machine,

Keep WeChat versions consistent on both sides,

Basically nothing else needs to be done.

So with GPT's help,

I wrote the following script backup_wechat.sh

```shell
#!/bin/bash
set -euo pipefail
BACKUP_SRC="$HOME/Library/Containers/com.tencent.xinWeChat"
BACKUP_DIR="/Volumes/workspace/weixin"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DEST="$BACKUP_DIR/wechat_backup_${TIMESTAMP}.tar.gz"
echo "🚀 Starting WeChat chat record backup..."
# Check WeChat process
if pgrep -f WeChat > /dev/null; then
  echo "⚠️ WeChat is detected running, please exit WeChat first (Cmd + Q), then run the script again."
  exit 1
fi
# Check data directory
if [ ! -d "$BACKUP_SRC" ]; then
  echo "❌ WeChat data directory not found: $BACKUP_SRC"
  exit 1
fi
# Check backup directory
if [ ! -d "$BACKUP_DIR" ]; then
  echo "❌ Backup directory does not exist: $BACKUP_DIR"
  exit 1
fi
echo "📦 Starting packaging... (Start time: $(date '+%F %T'))"
START_TIME=$(date +%s)
tar --exclude='.com.apple.containermanagerd.metadata.plist' --preserve-permissions -czf "$BACKUP_DEST" -C "$HOME/Library/Containers" com.tencent.xinWeChat
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "✅ Backup completed!"
echo "📁 File location: $BACKUP_DEST"
echo "⏱️ Time taken: ${DURATION} seconds ($(printf '%02d:%02d' $((DURATION/60)) $((DURATION%60))))"
```

Note: /Volumes/workspace/weixin is a NAS disk mounted locally,

After all, my computer doesn't have 170G of available space.

It's better to have an SSD portable hard drive,

But I don't have one on hand~

Restore script restore_wechat.sh

```shell
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/Volumes/workspace/weixin"
BACKUP_FILE=$(ls -t "$BACKUP_DIR"/wechat_backup_*.tar.gz 2>/dev/null | head -1 || true)
TARGET_DIR="$HOME/Library/Containers/com.tencent.xinWeChat"
if [ -z "$BACKUP_FILE" ]; then
  echo "❌ No backup file found, please confirm there is a WeChat backup file in $BACKUP_DIR directory."
  exit 1
fi
echo "🚀 Starting WeChat chat record restore..."
if pgrep -f WeChat > /dev/null; then
  echo "⚠️ WeChat is detected running, please exit WeChat first (Cmd + Q), then run the script again."
  exit 1
fi
echo "📦 Backup file to restore: $BACKUP_FILE"
echo "📦 File size: $(du -sh "$BACKUP_FILE" | awk '{print $1}')"
echo "📂 Starting decompression and overwrite old data... (Start time: $(date '+%F %T'))"
START_TIME=$(date +%s)
# Decompress, overwrite old content (do not delete original directory)
sudo tar -xzf "$BACKUP_FILE" -C "$HOME/Library/Containers"
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "✅ Restore completed!"
echo "⏱️ Time taken: ${DURATION} seconds ($(printf '%02d:%02d' $((DURATION/60)) $((DURATION%60))))"
echo "Please open WeChat to check if chat records are complete."
```

The whole process is even faster than mobile chat record backup and restore.

(After all, NAS transmission speed with full bandwidth is fast)

WeChat really....

In principle, Windows computers are similar,

But I basically don't fiddle with Windows anymore,

Figure it out yourself~
