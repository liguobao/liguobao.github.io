---
title: 跨云内网WireGuard 配置备忘录
description: linux wireguard
slug: wireguard-setup
date: 2025-06-08 08:00:00+0000
image: logo.jpg
categories:
  - TC
tags:
  - 2025
  - wireguard
  - linux
---


``` bash
#!/bin/bash
echo '开始执行wireguard-install.sh'
set -e

WG_DIR="/etc/wireguard"
KEY_DIR="/tmp/wg-keys"
WG_CONF="$WG_DIR/wg0.conf"
WG_NET_BASE="10.200.200"
WG_PORT=51820
NODE_LIST="$HOME/all-node.txt"

# === Step 0: 检查节点列表文件 ===
if [[ ! -f "$NODE_LIST" ]]; then
  echo "❌ 节点列表文件不存在：$NODE_LIST"
  exit 1
fi

mkdir -p "$WG_DIR"
mkdir -p "$KEY_DIR"
chmod 700 "$WG_DIR"

# === Step 1: 读取 IP 列表 ===
ALL_IPS=()
while read -r ip; do
  [[ -z "$ip" || "$ip" =~ ^# ]] && continue
  ALL_IPS+=("$ip")
done < "$NODE_LIST"
NODE_COUNT=${#ALL_IPS[@]}

# === Step 2: 获取公网IP函数，优先用 iinti.cn，失败用 ipify ===
get_public_ip() {
  ip=$(curl -s https://iinti.cn/conn/getPublicIp)
  if [[ -z "$ip" || "$ip" == "null" ]]; then
    ip=$(curl -s https://api.ipify.org)
  fi
  echo "$ip"
}

echo "准备获取公网 IP..."
MY_PUBLIC_IP=$(get_public_ip)
if [[ -z "$MY_PUBLIC_IP" ]]; then
  echo "❌ 无法获取公网 IP，退出"
  exit 1
fi
echo "获取到公网 IP：$MY_PUBLIC_IP"

MY_ID=0
for i in "${!ALL_IPS[@]}"; do
  if [[ "${ALL_IPS[$i]}" == "$MY_PUBLIC_IP" ]]; then
    MY_ID=$((i + 1))
    break
  fi
done

if [[ $MY_ID -eq 0 ]]; then
  echo "❌ 未在 all-node.txt 中找到当前公网IP：$MY_PUBLIC_IP"
  exit 1
fi

MY_WG_IP="${WG_NET_BASE}.${MY_ID}/24"
MY_KEY_NAME="node${MY_ID}.pub"

echo "  当前节点编号: $MY_ID"
echo "  公网 IP: $MY_PUBLIC_IP"
echo "  WireGuard IP: $MY_WG_IP"

# === Step 3: 安装 WireGuard（如未安装） ===
if ! command -v wg &>/dev/null; then
  echo "  安装 wireguard-tools..."
  apt update && apt install -y wireguard
fi

# === Step 4: 生成密钥 ===
PRIVATE_KEY_FILE="$WG_DIR/privatekey"
PUBLIC_KEY_FILE="$WG_DIR/publickey"

if [ ! -f "$PRIVATE_KEY_FILE" ]; then
  echo "  生成密钥对..."
  wg genkey | tee "$PRIVATE_KEY_FILE" | wg pubkey > "$PUBLIC_KEY_FILE"
fi

MY_PRIVATE_KEY=$(cat "$PRIVATE_KEY_FILE")
MY_PUBLIC_KEY=$(cat "$PUBLIC_KEY_FILE")

# === Step 5: 写本机公钥到本地目录 ===
echo "$MY_PUBLIC_KEY" > "$KEY_DIR/$MY_KEY_NAME"

# === Step 6: 自动将本机公钥 scp 到其他所有节点的 KEY_DIR ===
echo "  自动同步公钥到其他节点..."

for ip in "${ALL_IPS[@]}"; do
  # 跳过本机
  if [[ "$ip" == "$MY_PUBLIC_IP" ]]; then
    continue
  fi

  echo "⏳ 复制公钥到 $ip:$KEY_DIR"
  ssh -o ConnectTimeout=5 root@"$ip" "mkdir -p $KEY_DIR && chmod 700 $KEY_DIR"
  scp "$KEY_DIR/$MY_KEY_NAME" root@"$ip":"$KEY_DIR/"
done

# === Step 7: 等待所有节点的公钥文件都到齐 ===
echo "  等待所有节点公钥写入 $KEY_DIR ..."
while true; do
  FOUND_KEYS=$(ls "$KEY_DIR"/*.pub 2>/dev/null | wc -l)
  if [ "$FOUND_KEYS" -ge "$NODE_COUNT" ]; then
    echo "✅ 检测到所有 $NODE_COUNT 个节点的公钥"
    break
  fi
  echo -n "."
  sleep 2
done

# === Step 8: 生成 WireGuard 配置 ===
echo "⚙️ 生成 WireGuard 配置: $WG_CONF"

echo "[Interface]" > "$WG_CONF"
echo "Address = $MY_WG_IP" >> "$WG_CONF"
echo "ListenPort = $WG_PORT" >> "$WG_CONF"
echo "PrivateKey = $MY_PRIVATE_KEY" >> "$WG_CONF"
echo >> "$WG_CONF"

for i in "${!ALL_IPS[@]}"; do
  ID=$((i + 1))
  PEER_IP="${ALL_IPS[$i]}"
  PEER_WG_IP="${WG_NET_BASE}.${ID}/32"

  if [ "$ID" -eq "$MY_ID" ]; then
    continue
  fi

  PEER_KEY_FILE="$KEY_DIR/node${ID}.pub"
  if [ ! -f "$PEER_KEY_FILE" ]; then
    echo "❌ 缺少 $PEER_KEY_FILE，退出"
    exit 1
  fi

  PEER_KEY=$(cat "$PEER_KEY_FILE")
  echo "[Peer]" >> "$WG_CONF"
  echo "PublicKey = $PEER_KEY" >> "$WG_CONF"
  echo "AllowedIPs = $PEER_WG_IP" >> "$WG_CONF"
  echo "Endpoint = $PEER_IP:$WG_PORT" >> "$WG_CONF"
  echo "PersistentKeepalive = 25" >> "$WG_CONF"
  echo >> "$WG_CONF"
done

# === Step 9: 启动 WireGuard ===
echo "  启动 WireGuard..."
systemctl stop wg-quick@wg0 &>/dev/null || true
wg-quick up wg0
systemctl enable wg-quick@wg0

echo "✅ 启动成功，当前节点 WireGuard IP: ${WG_NET_BASE}.${MY_ID}"
echo "  查看状态: wg show"

```


## 前置条件
- 节点互相可以直接免密码登录
- 都有公网IP可达


折腾一下基本实现了跨云内网互通，延迟在可接受访问内。


( 毕竟跨大洋了。

之所以这样搞还是因为早些时间，

直接在k3s上折腾WireGuard 总有奇奇怪怪问题。

算了，自己搭吧。

现在看起来就稳定很多了。

也不在折腾k3s那边的奇怪配置了。

完事。