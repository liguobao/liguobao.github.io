---
title: "Best Edge Computing Cluster Solution: Tailscale + K3s (2026 Edition)"
slug: k3s-tailscale-cluster-2026
date: 2026-02-12 08:00:00+0000
image: logo.png
categories:
  - linux
tags:
  - k3s
  - tailscale
  - kubernetes
  - 2026
description: "A cross-cloud edge computing cluster solution based on self-hosted Tailscale control plane + K3s, with practical 8-node HA cluster deployment."
---


> PS: Holiday's coming, time for more tinkering!!

*Manual dog head emoji*

【Programming Tech Zone Disclaimer】

My last k3s article was written in 2025, using WireGuard for cross-cloud networking.

A year has passed, and WireGuard configuration is still too tedious.


—Every time you add a machine, you have to manually edit configs, add peers, adjust routes...

So this year, I decided to switch the underlying network entirely to Tailscale.

More specifically—self-hosting a Headscale control plane, without relying on the official SaaS.

This way, you can add as many nodes to the internal network as you want,

with a one-line command to join the network, no more manual WireGuard Peer configuration management.

At the same time, the k3s cluster has been upgraded from a humble 2-node version,


to an **8-node HA cluster with 4 Masters + 4 Edges**.

Small but complete.

Let's get to it.

## Overall Architecture

First, here's an architecture diagram for a global overview:

```
                    Internet
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    [Tencent Cloud] [Rainyun]    [Local]
    (Domestic)   (Singapore)   (Home/Office)
        │             │             │
        └─────────────┼─────────────┘
                      │
           Tailscale VPN (100.64.0.0/10)
          Headscale Self-hosted (ts.example.com)
          Authentik OIDC Login (auth.example.com)
                      │
        ┌──────────┬──────────┬──────────┐
        │          │          │          │
    [control1] [control2] [control3] [control4]
    vm-0-8     vm-16-12   vm-28-17   vm-0-15
   100.64.0.6 100.64.0.5 100.64.0.7 100.64.0.10
    (Primary)  (Member)   (Member)  (TS Control)
        │          │          │          │
        └──────────┴──────────┴──────────┘
                      │
        K3S HA Cluster (Flannel VXLAN over Tailscale)
        Pod: 10.42.0.0/16  |  Service: 10.43.0.0/16
                      │
        ┌─────────┬─────────┬─────────┐
        │         │         │         │
     [haru]  [lgb-amd]  [rainyun]  [hc1]
    Domestic   Local    Overseas  Domestic
    Ready ✅  Ready ✅  Ready ✅  Ready ✅
```

Current cluster state:

| Node Name | Role | Tailscale IP | Status | Notes |
|---------|------|-------------|------|------|
| vm-0-8-ubuntu-new | Master | 100.64.0.6 | Ready ✅ | Initial Master, --cluster-init |
| vm-16-12-ubuntu | Master | 100.64.0.5 | Ready ✅ | HA Master |
| vm-28-17-ubuntu | Master | 100.64.0.7 | Ready ✅ | HA Master |
| vm-0-15-ubuntu | Master | 100.64.0.10 | Ready ✅ | Headscale control plane node, NoSchedule |
| haru | Edge | 100.64.0.3 | Ready ✅ | Domestic node |
| lgb-amd-3700 | Edge | 100.64.0.4 | Ready ✅ | Local AMD host, 16GB |
| rainyun-ssh7pavp | Edge | 100.64.0.12 | Ready ✅ | Overseas Singapore node |
| hc1 | Edge | 100.64.0.9 | Ready ✅ | Domestic node |

8-node cluster running dozens of services, stable as a rock.

## Part I: Setting up Tailscale - Cloud Internal Network Cluster

### Why Not Manually Configure WireGuard Anymore?

In the 2025 article, I used the native WireGuard solution + k3s Flannel `wireguard-native` mode.

To be honest, WireGuard itself is very stable with good performance. But manually managing peers becomes a nightmare when you have multiple nodes:

- Every time you add a node, you need to update WireGuard configs on all other nodes
- Public key exchange and IP allocation are all manual
- If a machine's public IP changes, all peers need to be updated
- No unified management interface

So this time, I went straight to Tailscale.

### Why Self-host Headscale?

Tailscale's official SaaS service is certainly convenient, and the personal plan now supports up to 100 devices, which is more than enough. But there are a few issues:

1. **Login requires external network**—Tailscale login authentication uses Google/Microsoft/GitHub OAuth, basically unusable from mainland China without VPN, especially troublesome on servers
2. **Data control**—All node information is on someone else's servers, not reassuring
3. **Domestic access**—Tailscale's official coordination server is overseas, domestic nodes have unstable connections, occasionally slow to establish connections between nodes

So I chose [Headscale](https://github.com/juanfont/headscale)—an open-source Tailscale control plane implementation.

Combined with [Authentik](https://goauthentik.io/) for OIDC login, the experience is almost identical to official Tailscale, even more flexible.

### Headscale Deployment

**Server requirements**: A small 2C2G machine is enough, I'm using Tencent Cloud Lighthouse Ubuntu 24.04.

The core is a `docker-compose.yml` containing:

| Component | Description |
|------|------|
| Headscale v0.28.0 | Tailscale control plane |
| Authentik 2025.2.4 | OIDC username/password login |
| PostgreSQL 16 | Authentik database |
| Redis | Authentik cache |
| Nginx | HTTPS reverse proxy |

After deployment, two domains are ready:

- `https://ts.example.com` → Headscale control plane
- `https://auth.example.com` → Authentik login management

I won't expand on the detailed deployment process here (that's for another article), but the key points are:

1. **Use acme.sh + Nginx for HTTPS**, don't mess with fancy Traefik stuff
2. **Configure OIDC in Headscale**, with Issuer pointing to Authentik
3. **Use `100.64.0.0/10` IP range**, this is the CGNAT address range that won't conflict with internal networks

### Node Onboarding

After setting up Headscale, getting any machine on the network is a one-line command:

```bash
# Install Tailscale client
curl -fsSL https://tailscale.com/install.sh | sh

# Connect to self-hosted control plane
tailscale up --login-server=https://ts.example.com --hostname=your-hostname --accept-dns=false
```

The terminal will output a link, open it in a browser, it jumps to the Authentik login page, enter username and password, authorize—node is on the network.

That simple.

**Key recommendation**:

> **Don't rely on public cloud internal networks!**
> 
> Even if your Master nodes are in the same cloud, it's recommended to communicate uniformly through Tailscale.
> 
> The reason is simple: public cloud internal networks are black boxes. When you change machines or availability zones, internal IPs change.
> While Tailscale IPs are allocated by you and won't change.
> 
> All nodes, whether cloud-based, local, or on other cloud platforms, should uniformly access via Tailscale for a clean network architecture.

My current Tailscale network looks like this:

```
100.64.0.2   lgb-macbookair-m4   macOS     ← My dev machine
100.64.0.3   haru                linux     ← Edge node
100.64.0.4   lgb-amd-3700        linux     ← Local Edge node
100.64.0.5   vm-16-12-ubuntu     linux     ← K3s Master
100.64.0.6   vm-0-8-ubuntu-new   linux     ← K3s Master (Primary)
100.64.0.7   vm-28-17-ubuntu     linux     ← K3s Master
100.64.0.8   rainyun-vja2g92e    linux     ← Overseas node
100.64.0.9   hc1                 linux     ← Edge node
100.64.0.10  ts-headscale        linux     ← Headscale control plane + K3s Master
100.64.0.12  ipv6radar           linux     ← Overseas node
100.64.0.13  localhost           android   ← Phone can join too (for slacking off)
```

No matter where you are, `ping 100.64.0.6` works. This is what a proper internal network experience should be.

## Part II: K3s Cluster Setup - Based on Tailscale Network

### Core Principles

1. **All Master nodes in the same cloud region**—My 4 Masters are all in Tencent Cloud, low etcd sync latency
2. **Masters communicate using Tailscale IPs**—Don't rely on cloud internal network
3. **Edge nodes anywhere**—Home desktop, overseas VPS, office workstation, as long as Tailscale can reach
4. **Gateway nodes on high-bandwidth machines**—Ingress runs on lightweight cloud with sufficient traffic

### Installing Masters

Installation is actually straightforward, just follow the official docs: [https://docs.k3s.io/quick-start](https://docs.k3s.io/quick-start)

But there are a few **critical parameters** you must pay attention to.

**First Master (initialize cluster)**:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  sh -s - server \
  --cluster-init \
  --flannel-iface=tailscale0 \
  --node-ip=100.64.0.6 \
  --tls-san=100.64.0.6
```

**Subsequent Masters joining cluster**:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_TOKEN="your-token" \
  sh -s - server \
  --server https://100.64.0.6:6443 \
  --flannel-iface=tailscale0 \
  --node-ip=$(tailscale ip -4) \
  --tls-san=$(tailscale ip -4)
```

**Core parameter explanation**:

| Parameter | Why it's necessary |
|------|-----------|
| `--cluster-init` | First Master uses this, enables embedded etcd HA |
| `--flannel-iface=tailscale0` | **Most critical!** Makes Flannel VXLAN use Tailscale NIC, not physical NIC |
| `--node-ip=$(tailscale ip -4)` | Node IP uses Tailscale IP, ensures cross-cloud communication |
| `--tls-san=<ip>` | API Server certificate includes Tailscale IP |
| `INSTALL_K3S_MIRROR=cn` | Domestic mirror acceleration, overseas nodes don't need |

`--flannel-iface=tailscale0` is the hard-learned lesson after countless pitfalls.

Without this parameter, Flannel will default to using the physical NIC's IP (like public IP or cloud internal IP) to build VXLAN tunnels. The result is—**same-cloud nodes can communicate, cross-cloud nodes have complete Pod communication failure**.

With this parameter, Flannel's VXLAN tunnels all go through the Tailscale virtual NIC, cross-cloud Pod communication is perfect.

### Installing Edge Nodes

Edge nodes are Agents, even simpler.

**Domestic nodes**:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  K3S_URL="https://100.64.0.6:6443" \
  K3S_TOKEN="your-token" \
  INSTALL_K3S_MIRROR=cn \
  sh -s - agent \
  --node-ip=$(tailscale ip -4) \
  --flannel-iface=tailscale0 \
  --node-label="node.kubernetes.io/role=edge"
```

**Overseas nodes**:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="https://100.64.0.6:6443" \
  K3S_TOKEN="your-token" \
  sh -s - agent \
  --node-ip=$(tailscale ip -4) \
  --flannel-iface=tailscale0 \
  --node-label="node.kubernetes.io/role=edge"
```

The difference is domestic uses `rancher-mirror.rancher.cn`, overseas uses official `get.k3s.io`.

### Domestic Docker Image Pulling Issues

I need to specifically mention: **Docker/containerd image pulling in domestic environments is a big hassle.**

Docker Hub, ghcr.io, gcr.io and other image sources are basically semi-blocked in China. Some k3s built-in system component images (like `pause`, `coredns`, `metrics-server`) might not be pullable, causing nodes to stay NotReady.

Several solutions:

1. **Configure image accelerators**—Configure available domestic mirrors in `/etc/rancher/k3s/registries.yaml` (if you can still find live ones)
2. **Local export then import (recommended)**—Pull images on overseas nodes or machines that can pull normally, export and transfer to domestic nodes for import:
   ```bash
   # Export on overseas node
   ctr -n k8s.io images export pause.tar registry.k8s.io/pause:3.9
   
   # Transfer to domestic node
   scp pause.tar root@<domestic-node-IP>:/tmp/
   
   # Import on domestic node
   ctr -n k8s.io images import /tmp/pause.tar
   ```
3. **Self-host Harbor image registry**—If you have many nodes, it's recommended to set up a private image registry as a proxy cache, once and for all

My approach is to pre-pull needed images on overseas nodes, then `ctr images export` to export, transfer through Tailscale internal network to domestic nodes, and `ctr images import` to import. Though crude, it's stable and reliable.

Get the token on the first Master:

```bash
cat /var/lib/rancher/k3s/server/node-token
```

If everything's fine, you'll see new nodes online within seconds:

```bash
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES                       AGE    VERSION
vm-0-8-ubuntu-new    Ready    control-plane,etcd,master   30d    v1.32.3+k3s1
vm-16-12-ubuntu      Ready    control-plane,etcd,master   30d    v1.32.3+k3s1
vm-28-17-ubuntu      Ready    control-plane,etcd,master   30d    v1.32.3+k3s1
vm-0-15-ubuntu       Ready    control-plane,etcd,master   1d     v1.32.3+k3s1
haru                 Ready    <none>                      20d    v1.32.3+k3s1
lgb-amd-3700         Ready    <none>                      15d    v1.32.3+k3s1
rainyun-ssh7pavp     Ready    <none>                      10d    v1.32.3+k3s1
hc1                  Ready    <none>                      5d     v1.32.3+k3s1
```

All 8 nodes Ready.

Beautiful.

### About Gateway Nodes

Ingress traffic entry requires public IP + sufficient bandwidth.

My approach:

- **Gateway nodes use lightweight cloud servers**—Tencent Cloud Lighthouse, Alibaba Cloud Lighthouse, etc., monthly payment of tens of yuan, traffic package is sufficient
- **k3s built-in Traefik Ingress** runs on all nodes by default (DaemonSet), but only one or two nodes need to be exposed externally
- **Domain DNS resolves to these lightweight cloud public IPs**

Traffic path: `User request → Lightweight cloud public IP → Traefik Ingress → Service → Pod (can be on any node)`

Even if the Pod is on an overseas node, no problem—Flannel over Tailscale will route the traffic.

## Part III: Pitfall Chronicles

### Pitfall 1: Flannel Using Wrong NIC (Hard-learned Lesson)

**Symptom**: Edge node is Ready, but cross-node Pod communication fails completely.

**Cause**: Didn't add `--flannel-iface=tailscale0`, Flannel defaulted to public network NIC.

**Diagnosis**:
```bash
kubectl describe node rainyun-ssh7pavp | grep flannel
# See flannel.alpha.coreos.com/public-ip using public IP instead of 100.64.0.x
```

**Solution**: Uninstall k3s agent, reinstall with `--flannel-iface=tailscale0`.

This parameter is so important, say it three times:

1. `--flannel-iface=tailscale0`
2. `--flannel-iface=tailscale0`
3. `--flannel-iface=tailscale0`

### Pitfall 2: Unstable Tailscale Connection

A Vultr VPS had Tailscale connections dropping every few days.

Node kept bouncing between Ready and NotReady, kubelet frantically reporting `unable to update node status`.

Investigation revealed it was a VPN link issue, nothing to do with K3s.

**Solution**: First use taint to isolate, later just replaced the machine.

```bash
kubectl taint nodes vultr.guest node-problem=true:NoSchedule --overwrite
kubectl taint nodes vultr.guest node-problem=true:NoExecute --overwrite
```

Lesson: **Better to remove unstable nodes than struggle.** Replacing with a new machine is faster than troubleshooting network issues.

### Pitfall 3: Don't Run Business Pods on Low-spec Masters

My fourth Master (vm-0-15-ubuntu) only has 2G memory and also runs the Headscale control plane.

If you let it run business Pods, it'll OOM in minutes.

**Solution**: Add NoSchedule taint, only run control plane components + etcd.

```bash
kubectl taint nodes vm-0-15-ubuntu node-role.kubernetes.io/control-plane=:NoSchedule
```

2G memory running k3s control-plane + etcd + Headscale full stack, CPU and memory usage around 60%, holds up fine.

## Part IV: Troubleshooting Quick Reference

Node added but having issues? Check in this order:

```bash
# 1. Check node status
kubectl get nodes -o wide

# 2. Check node events
kubectl describe node <node-name>

# 3. Check Tailscale connectivity
tailscale ping <target-node-tailscale-IP>

# 4. Check Flannel configuration
kubectl describe node <node-name> | grep flannel

# 5. Check kubelet logs
ssh root@<node-IP> "journalctl -u k3s-agent -n 50"

# 6. Test cross-node Pod communication
kubectl run test --image=busybox --rm -it -- wget -qO- http://<other-node-PodIP>
```

90% of issues can be found in the first 4 steps.

## Part V: Summary

Compared to the 2025 solution, key changes in this upgrade:

| Item | 2025 | 2026 |
|------|---------|---------|
| Networking | WireGuard manual config | Tailscale (Headscale self-hosted) |
| Control plane | Official Tailscale / manual WG | Self-hosted Headscale + OIDC |
| Master count | 2 | 4 (HA) |
| Edge nodes | 0 | 4 |
| Total nodes | 2 | 8 |
| New node onboarding | Change lots of configs | One-line command |
| Cross-cloud communication | Flannel wireguard-native | Flannel VXLAN over Tailscale |
| Management complexity | High | Low |

**One-sentence solution summary**:

1. **Use Tailscale (Headscale self-hosted) for underlying network**—All nodes uniformly join, don't rely on any public cloud internal network
2. **K3s Masters in same cloud region**—Low etcd latency, stable control plane
3. **Edge nodes add freely**—Home machines, overseas VPS, office workstations, as long as Tailscale can reach
4. **Gateway nodes use lightweight cloud**—Cheap, sufficient bandwidth, fixed public IP

The entire cluster has been running for a month, stable as an old dog.

8/8 nodes all available (previously had 1 Vultr with network issues, already replaced).

Done.

Entire article completed in the study at dawn, Happy New Year everyone~

*Manual dog head emoji*

---

*Related links:*
- *K3s Official Docs: [https://docs.k3s.io/](https://docs.k3s.io/)*
- *Headscale: [https://github.com/juanfont/headscale](https://github.com/juanfont/headscale)*
- *Authentik: [https://goauthentik.io/](https://goauthentik.io/)*
- *Previous article: [Best Edge Computing Cluster Solution](https://liguobao.github.io/p/k3s-edge-computing-cluster/)*

---

*This article was written with AI assistance.*
