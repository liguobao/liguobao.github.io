---
title: Cross-Cloud Intranet WireGuard Configuration Memo
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
echo 'Starting wireguard-install.sh'
set -e

WG_DIR="/etc/wireguard"
KEY_DIR="/tmp/wg-keys"
WG_CONF="$WG_DIR/wg0.conf"
WG_NET_BASE="10.200.200"
WG_PORT=51820
NODE_LIST="$HOME/all-node.txt"

# === Step 0: Check node list file ===
if [[ ! -f "$NODE_LIST" ]]; then
  echo "❌ Node list file does not exist: $NODE_LIST"
  exit 1
fi

mkdir -p "$WG_DIR"
mkdir -p "$KEY_DIR"
chmod 700 "$WG_DIR"

# === Step 1: Read IP list ===
ALL_IPS=()
while read -r ip; do
  [[ -z "$ip" || "$ip" =~ ^# ]] && continue
  ALL_IPS+=("$ip")
done < "$NODE_LIST"
NODE_COUNT=${#ALL_IPS[@]}

# === Step 2: Get public IP function, prefer iinti.cn, fallback to ipify ===
get_public_ip() {
  ip=$(curl -s https://iinti.cn/conn/getPublicIp)
  if [[ -z "$ip" || "$ip" == "null" ]]; then
    ip=$(curl -s https://api.ipify.org)
  fi
  echo "$ip"
}

echo "Preparing to get public IP..."
MY_PUBLIC_IP=$(get_public_ip)
if [[ -z "$MY_PUBLIC_IP" ]]; then
  echo "❌ Unable to get public IP, exit"
  exit 1
fi
echo "Got public IP: $MY_PUBLIC_IP"

MY_ID=0
for i in "${!ALL_IPS[@]}"; do
  if [[ "${ALL_IPS[$i]}" == "$MY_PUBLIC_IP" ]]; then
    MY_ID=$((i + 1))
    break
  fi
done

if [[ $MY_ID -eq 0 ]]; then
  echo "❌ Current public IP not found in all-node.txt: $MY_PUBLIC_IP"
  exit 1
fi

MY_WG_IP="${WG_NET_BASE}.${MY_ID}/24"
MY_KEY_NAME="node${MY_ID}.pub"

echo "  Current node number: $MY_ID"
echo "  Public IP: $MY_PUBLIC_IP"
echo "  WireGuard IP: $MY_WG_IP"

# === Step 3: Install WireGuard (if not installed) ===
if ! command -v wg &>/dev/null; then
  echo "  Installing wireguard-tools..."
  apt update && apt install -y wireguard
fi

# === Step 4: Generate keys ===
PRIVATE_KEY_FILE="$WG_DIR/privatekey"
PUBLIC_KEY_FILE="$WG_DIR/publickey"

if [ ! -f "$PRIVATE_KEY_FILE" ]; then
  echo "  Generating key pair..."
  wg genkey | tee "$PRIVATE_KEY_FILE" | wg pubkey > "$PUBLIC_KEY_FILE"
fi

MY_PRIVATE_KEY=$(cat "$PRIVATE_KEY_FILE")
MY_PUBLIC_KEY=$(cat "$PUBLIC_KEY_FILE")

# === Step 5: Write local public key to local directory ===
echo "$MY_PUBLIC_KEY" > "$KEY_DIR/$MY_KEY_NAME"

# === Step 6: Automatically scp local public key to all other nodes' KEY_DIR ===
echo "  Automatically sync public key to other nodes..."

for ip in "${ALL_IPS[@]}"; do
  # Skip local machine
  if [[ "$ip" == "$MY_PUBLIC_IP" ]]; then
    continue
  fi

  echo "⏳ Copy public key to $ip:$KEY_DIR"
  ssh -o ConnectTimeout=5 root@"$ip" "mkdir -p $KEY_DIR && chmod 700 $KEY_DIR"
  scp "$KEY_DIR/$MY_KEY_NAME" root@"$ip":"$KEY_DIR/"
done

# === Step 7: Wait for all nodes' public key files to be ready ===
echo "  Waiting for all node public keys to be written to $KEY_DIR ..."
while true; do
  FOUND_KEYS=$(ls "$KEY_DIR"/*.pub 2>/dev/null | wc -l)
  if [ "$FOUND_KEYS" -ge "$NODE_COUNT" ]; then
    echo "✅ Detected all $NODE_COUNT nodes' public keys"
    break
  fi
  echo -n "."
  sleep 2
done

# === Step 8: Generate WireGuard configuration ===
echo "⚙️ Generating WireGuard configuration: $WG_CONF"

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
    echo "❌ Missing $PEER_KEY_FILE, exit"
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

# === Step 9: Start WireGuard ===
echo "  Starting WireGuard..."
systemctl stop wg-quick@wg0 &>/dev/null || true
wg-quick up wg0
systemctl enable wg-quick@wg0

echo "✅ Started successfully, current node WireGuard IP: ${WG_NET_BASE}.${MY_ID}"
echo "  Check status: wg show"
```

## Prerequisites

- Nodes can log in to each other without password
- All have reachable public IPs

After some fiddling, basically achieved cross-cloud intranet connectivity, latency within acceptable range.

(After all, it's across oceans.

The reason for this setup is that earlier,

Directly fiddling with WireGuard on k3s always had strange issues.

Forget it, set it up myself.

Now it looks much more stable.

No more fiddling with strange configurations on k3s side.

Done.
