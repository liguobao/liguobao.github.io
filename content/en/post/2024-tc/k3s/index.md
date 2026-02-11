---
title: "[k3s] Best Edge Computing Cluster Solution of the Year"
description: "This year's best edge computing cluster solution"
slug: k3s-edge-computing-cluster
date: 2024-05-01 08:00:00+0000
image: k3s.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---

PS: Not working on Labor Day — just kidding!

Manual dog head ~



[Disclaimer on programming technical area]

One of our servers was about to expire and the IDC quoted 2500 for renewal.
They said the price was the same last year. Fair enough — we did pay last year.
But this year isn’t easy.

So I looked for cloud deals and bought machines quickly. I decided to build a cluster.
I chose k3s.


I had used other edge solutions like SuperEdge before, but those projects stalled.
Many open-source projects from large companies in China seem to be KPI-driven and get abandoned.
K3s has been more actively maintained by the Rancher team.

Why is it called K3s?

The joke says we want Kubernetes that uses half the memory. Kubernetes is abbreviated to K8s. Half of 10 letters is 5, so the joke abbreviates it to K3s. K3s doesn’t have an official long name or pronunciation. See the docs: <https://docs.k3s.io/zh/>.

Small but powerful.

I handed the task to Haige; he spent an afternoon getting the cluster up. We migrated services that night and it took about three and a half hours.
We accidentally lost a Redis cache (my mistake), but the cluster went online.

Two cheap nodes running:

vm-16-12-ubuntu Ready 42d v1.28.7+k3s1
vm-28-17-ubuntu Ready 42d v1.28.7+k3s1

I’ve been busy with GPT2077 recently and left little time for the machines.
Now that things are calm and a server renewal came up, I started tinkering again.


Installation follows the official docs: <https://docs.k3s.io/zh/quick-start>.

Example (China mirror):

```sh
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -
```


Notes and gotchas

1) After running the script, the installer creates a k3s.service and installs the k3s CLI.

Whether a node runs as a server or an agent depends on k3s.service configuration.
By default the system starts as a server; adjust the service arguments to run as an agent or use other options.



## 2) Edge networking

My servers are two cloud hosts in the same LAN, so configuration is simple. But edge nodes may sit behind NAT or in other clouds, making direct internal networking impossible.

WireGuard is a good option for cross-location networking.

What is WireGuard?

WireGuard is a simple, fast, and secure VPN that uses modern cryptography. It aims to be easier and slimmer than traditional VPNs and runs on many platforms including Linux, Windows, macOS, BSD, iOS, and Android. Reference: <https://zhuanlan.zhihu.com/p/108365587>

I first looked at Netmaker (but it’s a paid project with conceptual complexity). Then I found Yang Bin’s article about using WireGuard to build K3s across clouds and adopted the approach.

The plan:

- Use WireGuard for site-to-site networking
- Configure k3s Flannel to use wireguard-native
- Restart and verify the cluster



## Install WireGuard

### Debian / Ubuntu

```sh
apt install wireguard resolvconf -y

# Enable IPv4 forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

### CentOS 7

```sh
sudo yum install epel-release elrepo-release -y
sudo yum install yum-plugin-elrepo -y
sudo yum install kmod-wireguard wireguard-tools systemd-resolved -y

# Enable IPv4 forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

CentOS may need a reboot after kernel module updates.


## k3s Flannel -> wireguard-native

Edit the k3s systemd service and restart k3s:

```sh
sudo vi /etc/systemd/system/k3s.service
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

Example `ExecStart` snippet for `/etc/systemd/system/k3s.service`:

```sh
ExecStart=/usr/local/bin/k3s \
    server \
    --node-external-ip <PUBLIC_IP> \
    --flannel-backend wireguard-native \
    --flannel-external-ip <PUBLIC_IP> \
    '--tls-san' \
    '<PUBLIC_IP>' \
    '--node-name' \
    'vm-16-12-ubuntu' \
    '--server' \
    'https://<INTERNAL_IP>:6443' \
```

If there are no errors, restart the service and check logs.

Next, configure agent nodes (edge nodes).

```sh
# Agent example
k3s agent --node-name zj-hc1 \
  --lb-server-port 5443 \
  --server https://<SERVER_PUBLIC_IP>:6443 \
  --token="<YOUR_TOKEN>"
```

I used `--lb-server-port 5443` because `6443` was taken. This is optional.

More agent options: <https://docs.rancher.cn/docs/k3s/installation/install-options/agent-config/_index/>

After the agent joins, modify `/etc/systemd/system/k3s.service` on the agent and restart k3s:

```sh
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

Example agent `ExecStart` core options:

```sh
ExecStart=/usr/local/bin/k3s \
    agent --node-name zj-hc2 \
    --lb-server-port 5443 \
    --node-ip 10.42.5.1  \
    --node-external-ip <PUBLIC_IP> \
    --server https://<YOUR_SERVER_IP>:6443 \
    --token="<YOUR_TOKEN>"
```


That’s it — the cluster should be up.

I tested basic services and the latency is acceptable for non-latency-sensitive workloads.

Written on a speeding train — still worth a thumbs up.

Manual dog head ~

Happy holidays!
