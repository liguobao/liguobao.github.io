---
title: "2206年最佳边缘计算集群方案：Tailscale + k3s"
slug: k3s-tailscale-cluster-2026
date: 2026-02-11 08:00:00+0000
image: logo.png
categories:
  - linux
tags:
  - k3s
  - tailscale
  - kubernetes
  - 2026
description: "基于 Tailscale 自建控制面 + K3s 的跨云边缘计算集群方案，实战 8 节点 HA 集群搭建。"
---


> PS：马上放假了，再折腾一下！！！

手动狗头~

【编程技术专区劝退提示】

上一篇 k3s 文章已经是 2025 年了，当时用的是 WireGuard 做跨云组网。

一年过去了，WireGuard 配置搞来搞去还是太累了


——每加一台机器都得手动改配置、互加 Peer、调路由……

于是今年决定把底层网络全换成 Tailscale。

更具体一点——自建 Headscale 控制面，不依赖官方 SaaS。

这样一来，整个内网想加多少节点就加多少节点，

一行命令入网，不用再手动管 WireGuard Peer 配置了。

同时，k3s 集群从 2 节点寒酸版，


升级到了 **4 Master + 4 Edge 的 8 节点 HA 集群**。

麻雀虽小，五脏俱全。

直接干。

## 整体架构

先放一张架构图，全局概览：

```
                    互联网
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    [腾讯云]       [Rainyun]      [本地]
    (国内多台)    (海外新加坡)   (家庭/办公)
        │             │             │
        └─────────────┼─────────────┘
                      │
           Tailscale VPN (100.64.0.0/10)
          Headscale 自建控制面 (ts.example.com)
          Authentik OIDC 登录 (auth.example.com)
                      │
        ┌──────────┬──────────┬──────────┐
        │          │          │          │
    [control1] [control2] [control3] [control4]
    vm-0-8     vm-16-12   vm-28-17   vm-0-15
   100.64.0.6 100.64.0.5 100.64.0.7 100.64.0.10
    (Primary)  (Member)   (Member)  (TS控制面)
        │          │          │          │
        └──────────┴──────────┴──────────┘
                      │
        K3S HA 集群 (Flannel VXLAN over Tailscale)
        Pod: 10.42.0.0/16  |  Service: 10.43.0.0/16
                      │
        ┌─────────┬─────────┬─────────┐
        │         │         │         │
     [haru]  [lgb-amd]  [rainyun]  [hc1]
      国内      本地      海外      国内
    Ready ✅  Ready ✅  Ready ✅  Ready ✅
```

当前集群实际状态：

| 节点名称 | 角色 | Tailscale IP | 状态 | 说明 |
|---------|------|-------------|------|------|
| vm-0-8-ubuntu-new | Master | 100.64.0.6 | Ready ✅ | 初始 Master, --cluster-init |
| vm-16-12-ubuntu | Master | 100.64.0.5 | Ready ✅ | HA Master |
| vm-28-17-ubuntu | Master | 100.64.0.7 | Ready ✅ | HA Master |
| vm-0-15-ubuntu | Master | 100.64.0.10 | Ready ✅ | Headscale 控制面节点, NoSchedule |
| haru | Edge | 100.64.0.3 | Ready ✅ | 国内节点 |
| lgb-amd-3700 | Edge | 100.64.0.4 | Ready ✅ | 本地 AMD 主机, 16GB |
| rainyun-ssh7pavp | Edge | 100.64.0.12 | Ready ✅ | 海外新加坡节点 |
| hc1 | Edge | 100.64.0.9 | Ready ✅ | 国内节点 |

8 个节点的集群，跑着十几个服务，稳得一比。

## 一、先搞 Tailscale：云端内网集群

### 为什么不用 WireGuard 手动组网了？

2025 年写的那篇文章里，我用的是 WireGuard 原生方案 + k3s Flannel `wireguard-native` 模式。

说实话，WireGuard 本身很稳，性能也好。但手动管理 Peer 这件事，在节点多了之后简直是噩梦：

- 每加一个节点，其他节点的 WireGuard 配置都要改
- 公钥交换、IP 分配全靠手动
- 某台机器换了公网 IP，所有 Peer 都得更新
- 没有统一的管理界面

所以，这次直接上 Tailscale。

### 为什么自建 Headscale？

Tailscale 官方的 SaaS 服务当然好用，个人版现在支持最多 100 台设备，数量上完全够用。但有几个问题：

1. **登录需要外网**——Tailscale 的登录认证走的是 Google/Microsoft/GitHub OAuth，国内裸连基本没戏，每次 `tailscale up` 都得先翻墙授权，服务器端更是麻烦
2. **数据控制**——所有节点信息都在别人那，心里不踏实
3. **国内访问**——Tailscale 官方的协调服务器（coordination server）在海外，国内节点连接不太稳定，偶尔会出现节点间建连慢的问题

所以我选择了 [Headscale](https://github.com/juanfont/headscale)——开源的 Tailscale 控制面实现。

配合 [Authentik](https://goauthentik.io/) 做 OIDC 登录，体验和官方 Tailscale 几乎一致，甚至更灵活。

### Headscale 部署

**服务器要求**：一台 2C2G 的小机器就够了，我用的是腾讯云轻量 Ubuntu 24.04。

核心就是一个 `docker-compose.yml`，包含：

| 组件 | 说明 |
|------|------|
| Headscale v0.28.0 | Tailscale 控制面 |
| Authentik 2025.2.4 | OIDC 账号密码登录 |
| PostgreSQL 16 | Authentik 数据库 |
| Redis | Authentik 缓存 |
| Nginx | HTTPS 反向代理 |

部署完之后，两个域名就位：

- `https://ts.example.com` → Headscale 控制面
- `https://auth.example.com` → Authentik 登录管理

详细的部署流程这里不展开了（那是另一篇文章的事），核心要点：

1. **HTTPS 用 acme.sh + Nginx**，别搞 Traefik 那些花里胡哨的
2. **Headscale 配置 OIDC**，Issuer 指向 Authentik
3. **IP 段用 `100.64.0.0/10`**，这是 CGNAT 地址段，不会和内网冲突

### 节点入网

装好 Headscale 之后，任何一台机器入网就是一行命令的事：

```bash
# 安装 Tailscale 客户端
curl -fsSL https://tailscale.com/install.sh | sh

# 连接到自建控制面
tailscale up --login-server=https://ts.example.com --hostname=你的主机名 --accept-dns=false
```

终端会吐出一个链接，浏览器打开，跳到 Authentik 登录页，输入账号密码，授权完成——节点入网。

就这么简单。

**关键建议**：

> **不要依赖公有云内网！**
> 
> 即使你的 Master 节点在同一个云，也建议统一走 Tailscale 通信。
> 
> 原因很简单：公有云内网是个黑盒，哪天换个机器、换个可用区，内网 IP 就变了。
> 而 Tailscale IP 是你自己分配的，不会变。
> 
> 所有节点，不管是云端、本地、还是其他云平台，统一用 Tailscale 接入，网络架构干净利落。

当前我的 Tailscale 网络长这样：

```
100.64.0.2   lgb-macbookair-m4   macOS     ← 我的开发机
100.64.0.3   haru                linux     ← Edge 节点
100.64.0.4   lgb-amd-3700        linux     ← 本地 Edge 节点
100.64.0.5   vm-16-12-ubuntu     linux     ← K3s Master
100.64.0.6   vm-0-8-ubuntu-new   linux     ← K3s Master (Primary)
100.64.0.7   vm-28-17-ubuntu     linux     ← K3s Master
100.64.0.8   rainyun-vja2g92e    linux     ← 海外节点
100.64.0.9   hc1                 linux     ← Edge 节点
100.64.0.10  ts-headscale        linux     ← Headscale 控制面 + K3s Master
100.64.0.12  ipv6radar           linux     ← 海外节点
100.64.0.13  localhost           android   ← 手机也能入网（摸鱼用）
```

不管在哪，`ping 100.64.0.6` 就能通。这才是正确的内网体验。

## 二、K3s 集群搭建：基于 Tailscale 网络

### 核心原则

1. **Master 节点全部放在云端同一家云**——我的 4 台 Master 都在腾讯云，etcd 同步延迟低
2. **Master 之间也用 Tailscale IP 通信**——不依赖云内网
3. **Edge 节点随意**——家里的台式机、海外的 VPS、公司的工作站，只要 Tailscale 能通就行
4. **网关节点用大流量机器**——Ingress 跑在轻量云上，流量够用

### 安装 Master

安装其实没什么花活，看官方文档就好：[https://docs.k3s.io/zh/quick-start](https://docs.k3s.io/zh/quick-start)

但有几个**关键参数**必须注意。

**第一个 Master（初始化集群）**：

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  sh -s - server \
  --cluster-init \
  --flannel-iface=tailscale0 \
  --node-ip=100.64.0.6 \
  --tls-san=100.64.0.6
```

**后续 Master 加入集群**：

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_TOKEN="你的Token" \
  sh -s - server \
  --server https://100.64.0.6:6443 \
  --flannel-iface=tailscale0 \
  --node-ip=$(tailscale ip -4) \
  --tls-san=$(tailscale ip -4)
```

**核心参数说明**：

| 参数 | 为什么必须 |
|------|-----------|
| `--cluster-init` | 第一个 Master 用，启用嵌入式 etcd HA |
| `--flannel-iface=tailscale0` | **最关键！** 让 Flannel VXLAN 走 Tailscale 网卡，不是物理网卡 |
| `--node-ip=$(tailscale ip -4)` | 节点 IP 用 Tailscale IP，保证跨云通信 |
| `--tls-san=<ip>` | API Server 证书包含 Tailscale IP |
| `INSTALL_K3S_MIRROR=cn` | 国内镜像加速，海外节点不需要 |

`--flannel-iface=tailscale0` 这个参数是踩了无数坑之后的血泪教训。

如果不加这个参数，Flannel 会默认用物理网卡的 IP（比如公网 IP 或者云内网 IP）来建 VXLAN 隧道。结果就是——**同云的节点能通，跨云的节点 Pod 全部不通**。

加了这个参数之后，Flannel 的 VXLAN 隧道全部走 Tailscale 虚拟网卡，跨云 Pod 通信完美。

### 安装 Edge 节点

Edge 节点就是 Agent，更简单。

**国内节点**：

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  K3S_URL="https://100.64.0.6:6443" \
  K3S_TOKEN="你的Token" \
  INSTALL_K3S_MIRROR=cn \
  sh -s - agent \
  --node-ip=$(tailscale ip -4) \
  --flannel-iface=tailscale0 \
  --node-label="node.kubernetes.io/role=edge"
```

**海外节点**：

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="https://100.64.0.6:6443" \
  K3S_TOKEN="你的Token" \
  sh -s - agent \
  --node-ip=$(tailscale ip -4) \
  --flannel-iface=tailscale0 \
  --node-label="node.kubernetes.io/role=edge"
```

差别就是国内用 `rancher-mirror.rancher.cn`，海外用官方 `get.k3s.io`。

### 国内 Docker 镜像拉取问题

这里要特别提一下：**国内环境下 Docker / containerd 拉取镜像是个大麻烦。**

Docker Hub、ghcr.io、gcr.io 这些镜像源在国内基本处于半墙状态，k3s 内置的一些系统组件镜像（比如 `pause`、`coredns`、`metrics-server`）可能拉不下来，导致节点一直 NotReady。

几个解决思路：

1. **配置镜像加速器**——在 `/etc/rancher/k3s/registries.yaml` 中配置国内可用的 mirror（如果你还能找到活着的）
2. **本地导出再导入（推荐）**——在海外节点或者能正常拉取的机器上先把镜像拉下来，导出后传到国内节点导入：
   ```bash
   # 在海外节点导出
   ctr -n k8s.io images export pause.tar registry.k8s.io/pause:3.9
   
   # 传到国内节点
   scp pause.tar root@<国内节点IP>:/tmp/
   
   # 在国内节点导入
   ctr -n k8s.io images import /tmp/pause.tar
   ```
3. **自建 Harbor 镜像仓库**——如果节点多，建议搞一个私有镜像仓库做 proxy cache，一劳永逸

我的做法是在海外节点上提前拉好需要的镜像，然后 `ctr images export` 导出，通过 Tailscale 内网传到国内节点再 `ctr images import` 导入。虽然笨了点，但胜在稳定可靠。

Token 在第一个 Master 上获取：

```bash
cat /var/lib/rancher/k3s/server/node-token
```

没什么问题的话，几秒钟后就能看到新节点上线：

```bash
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES                       AGE    VERSION
vm-0-8-ubuntu-new    Ready    control-plane,etcd,master   30d    v1.34.3+k3s3
vm-16-12-ubuntu      Ready    control-plane,etcd,master   30d    v1.34.3+k3s3
vm-28-17-ubuntu      Ready    control-plane,etcd,master   30d    v1.34.3+k3s3
vm-0-15-ubuntu       Ready    control-plane,etcd,master   1d     v1.34.3+k3s3
haru                 Ready    <none>                      20d    v1.34.3+k3s3
lgb-amd-3700         Ready    <none>                      15d    v1.34.3+k3s3
rainyun-ssh7pavp     Ready    <none>                      10d    v1.34.3+k3s3
hc1                  Ready    <none>                      5d     v1.34.3+k3s3
```

8 个节点全部 Ready。

美。

### 关于网关节点

Ingress 流量入口需要公网 IP + 足够的带宽。

我的做法是：

- **网关节点用轻量云服务器**——腾讯云轻量、阿里云轻量之类的，月付几十块，流量包够用
- **k3s 自带的 Traefik Ingress** 默认会跑在所有节点上（DaemonSet），但实际对外暴露的只需要一两个节点
- **域名 DNS 解析到这些轻量云的公网 IP** 就行

流量路径：`用户请求 → 轻量云公网IP → Traefik Ingress → Service → Pod（可能在任意节点上）`

Pod 在海外节点上也没关系，Flannel over Tailscale 会把流量打通。

## 三、踩坑实录

### 坑 1：Flannel 用错网卡（血泪教训）

**症状**：Edge 节点 Ready 了，但跨节点 Pod 通信全挂。

**原因**：没加 `--flannel-iface=tailscale0`，Flannel 默认选了公网网卡。

**诊断**：
```bash
kubectl describe node rainyun-ssh7pavp | grep flannel
# 看到 flannel.alpha.coreos.com/public-ip 用的是公网 IP 而不是 100.64.0.x
```

**解决**：卸载 k3s agent，重装时加上 `--flannel-iface=tailscale0`。

这个参数真的太重要了，说三遍：

1. `--flannel-iface=tailscale0`
2. `--flannel-iface=tailscale0`
3. `--flannel-iface=tailscale0`

### 坑 2：Tailscale 连接不稳定

某台 Vultr 的 VPS，Tailscale 连接三天两头断。

节点在 Ready 和 NotReady 之间反复横跳，kubelet 疯狂报 `unable to update node status`。

排查下来发现是 VPN 链路问题，跟 K3s 没关系。

**解决方案**：先用 taint 隔离，后面直接换掉了这台机器。

```bash
kubectl taint nodes vultr.guest node-problem=true:NoSchedule --overwrite
kubectl taint nodes vultr.guest node-problem=true:NoExecute --overwrite
```

教训：**不稳定的节点宁可不要，别死磕。** 换一台新机器比排查网络问题快得多。

### 坑 3：低配 Master 别跑业务 Pod

我的第四台 Master（vm-0-15-ubuntu）只有 2G 内存，同时还跑着 Headscale 控制面。

如果让它跑业务 Pod，分分钟 OOM。

**解决**：加 NoSchedule taint，只跑控制平面组件 + etcd。

```bash
kubectl taint nodes vm-0-15-ubuntu node-role.kubernetes.io/control-plane=:NoSchedule
```

2G 内存跑 k3s control-plane + etcd + Headscale 全家桶，CPU 和内存占用大约 60%，稳得住。

## 四、故障排查速查

节点加完之后出问题？按这个顺序查：

```bash
# 1. 看节点状态
kubectl get nodes -o wide

# 2. 看节点事件
kubectl describe node <节点名>

# 3. 看 Tailscale 连通性
tailscale ping <目标节点Tailscale IP>

# 4. 看 Flannel 配置对不对
kubectl describe node <节点名> | grep flannel

# 5. 看 kubelet 日志
ssh root@<节点IP> "journalctl -u k3s-agent -n 50"

# 6. 测试跨节点 Pod 通信
kubectl run test --image=busybox --rm -it -- wget -qO- http://<其他节点PodIP>
```

90% 的问题都是前 4 步能查出来的。

## 五、总结

对比 2025 年的方案，这次升级的核心变化：

| 项目 | 2025 年 | 2026 年 |
|------|---------|---------|
| 组网方案 | WireGuard 手动配置 | Tailscale (Headscale 自建) |
| 控制面 | 官方 Tailscale / 手动 WG | 自建 Headscale + OIDC |
| Master 数量 | 2 | 4 (HA) |
| Edge 节点 | 0 | 4 |
| 总节点数 | 2 | 8 |
| 新节点接入 | 改一堆配置 | 一行命令 |
| 跨云通信 | Flannel wireguard-native | Flannel VXLAN over Tailscale |
| 管理复杂度 | 高 | 低 |

**一句话总结方案**：

1. **用 Tailscale（Headscale 自建）做底层网络**——所有节点统一接入，不依赖任何公有云内网
2. **K3s Master 放在同一家云的同一地域**——etcd 延迟低，控制平面稳定
3. **Edge 节点随便加**——家里的机器、海外的 VPS、办公室的工作站，只要 Tailscale 能通就行
4. **网关节点用轻量云**——便宜、带宽够、公网 IP 固定

整个集群跑了一个月，稳如老狗。

8/8 节点全部可用（之前有 1 台 Vultr 网络不行，已经换掉了）。

完事。

全文在凌晨的书房里完成，祝大家新年快乐~

手动狗头~

---

*相关链接：*
- *K3s 官方文档：[https://docs.k3s.io/zh/](https://docs.k3s.io/zh/)*
- *Headscale：[https://github.com/juanfont/headscale](https://github.com/juanfont/headscale)*
- *Authentik：[https://goauthentik.io/](https://goauthentik.io/)*
- *上一篇：[【k3s】年度最佳边缘计算集群方案](https://liguobao.github.io/p/k3s-edge-computing-cluster/)*

---

*本文使用 AI 辅助写作。*
