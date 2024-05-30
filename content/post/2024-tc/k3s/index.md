---
title: 【k3s】年度最佳边缘计算集群方案
description: 今年最佳边缘计算集群方案
slug: k3s-edge-computing-cluster
date: 2024-05-1 08:00:00+0000
image: gqj.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---


PS：劳动节不劳动，犯法的！

手动狗头~





【编程技术专区劝退提示】

最近有台服务器要到期了，问了下IDC机房续费，报价说2500。

供应商说去年也是这个价格。

道理倒没错，确实去年是付了钱的。

只是....今年嘛，日子都不太好过。

算了，去壕各种云羊毛算求了。

很快就买了机器，想着一不做二不休——想搞集群方案。

于是，盯上了k3s。





之前用过SuperEdge之类的边缘集群方案，没有release就再用了。

然后今年他们团队...基本属于不更新状态了。

果然国内大厂开源都基本是KPI项目，我再信他们我就是狗。

k3s一直都有点关注，

毕竟rancher这公司在17、18年可是我们部署k8s集群的救星，

他们开源的项目一直都算是靠谱的。

为什么叫 K3s?

我们希望安装的 Kubernetes 只占用一半的内存。Kubernetes 是一个 10 个字母的单词，简写为 K8s。
Kubernetes 的一半就是一个 5 个字母的单词，因此简写为 K3s。
K3s 没有全称，也没有官方的发音。https://docs.k3s.io/zh/


对嘛，多有意思。












同时，麻雀虽小，应用尽有。



直接干。

（然后我把这活扔给了海哥，海哥花了一下午把集群跑起来了。

（接着我们晚上开始迁移服务，折腾了三个半小时。

（期间丢失了一次Redsi缓存数据（我误删

（很“顺利”就上线了

vm-16-12-ubuntu Ready 42d v1.28.7+k3s1

vm-28-17-ubuntu Ready 42d v1.28.7+k3s1

两个节点的寒酸集群~



这阵子忙着搞GPT2077嘛，基本机器就没咋搞了。

（插播广告：GPT2077：支持论文解读、邀请好友升级~

这两天没那么忙了，又遇上服务器到期续费，也是想着开始折腾下了。

开搞。





安装什么的，看官网文档就好。



https://docs.k3s.io/zh/quick-start





curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh \


cn user

curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -




这里主要是讲几个注意事项。



一、安装事项



上面的脚本安装在本地安装完成之后，

默认直接装了一个k3s.service，同时也把k3s cli装好了，








至于这个节点是个server还是个agent，

主要就看自己怎么去改k3s.service的配置了。










默认这个是启动server，我们可以根据需求启动成agent，

或者使用其他的参数配置。



二、边缘计算网络



我这边的Server是两台云端的服务器，他们在同一个局域网互通，配置起来很简单。

但是其他的机器，可能是在没有公网IP的本地，也可能是其他云的单机器。这样一来，肯定就没办法直接做内网通讯了。

查了一些资料之后发现，WireGuard 是一个很不错的方案。

什么是 WireGuard？
WireGuard 是一个易于配置、快速且安全的开源 VPN，它利用了最新的加密技术。目的是提供一种更快、更简单、更精简的通用 VPN，它可以轻松地在树莓派这类低端设备到高端服务器上部署。
WireGuard 最初是为 Linux 开发的，但现在可用于 Windows、macOS、BSD、iOS 和 Android。它仍在活跃开发中。
https://zhuanlan.zhihu.com/p/108365587


最早参考的是：

- 使用 k3s 和 WireGuard 网络部署 Kubernetes 集群：https://einverne.github.io/post/2021/12/kubernetes-cluster-multiple-clouds-using-k3s-netmaker-wireguard.html

折腾下来发现，Netmaker 是付费项目，配置起来还有一些概念需要理解。

反正第一次并没有成功，而且这是21年的博客文章了,有点过时也有可能。



没死心，继续找资料折腾。



于是看到了 【杨斌】大佬的专栏文章：

- Kubernetes 入门到实践：借助 WireGuard 跨云搭建 K3s 集群环境：https://zhuanlan.zhihu.com/p/584998631

看了一圈下来发现应该是适合我这边的。

由于我的云主机分布在不同的云服务商，所以不能通过服务商的内网环境互相访问，这里需要借助 WireGuard 完成异地组网。由于 K3s 通过 Flannel 已经集成了 WireGuard，所以我们可以通过一些简单的配置即可轻松完成组网。https://zhuanlan.zhihu.com/p/584998631


所以方案大体是：

- WireGuard 工具

- k3s Flannel切换成 wireguard-native

- 重启集群验证



安装 WireGuard



# 安装 WireGuard 
apt install wireguard resolvconf -y

# 开启 IPV4 IP转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p


或者

# centos7
sudo yum install epel-release elrepo-release -y;
sudo yum install yum-plugin-elrepo -y;
sudo yum install kmod-wireguard wireguard-tools systemd-resolved -y;


# 开启 IPV4 IP转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
Centos 配置好了之后可能需要重启一下机器，毕竟内核文件也更新了。



k3s Flannel切换

# 修改service 启动命令
sudo vi /etc/systemd/system/k3s.service

sudo systemctl daemon-reload
sudo systemctl restart k3s
# /etc/systemd/system/k3s.service

ExecStart=/usr/local/bin/k3s \
    server \
  --node-external-ip 公网IP \
    --flannel-backend wireguard-native \
    --flannel-external-ip 公网Ip \
  '--tls-san' \
  '公网Ip' \
  '--node-name' \
  'vm-16-12-ubuntu' \
  '--server' \
  'https://内网IP:6443' \


大概如上。

没有问题的话，重启这个服务就好。

有问题就去看日志了，遇神拜神~

server 没什么问题的话，接下来就到了折腾Agent的时候了。

Agent也就是我们的边缘节点了。





Agent 节点配置

依旧还是改k3s.service



首先，你得先知道 wireguard 会给你哪个地址。

我一般都是先直接用k3s agent 命令看下运行输出，有问题也能马上看到。

k3s agent --node-name zj-hc1 \
--lb-server-port 5443 \
--server https://server的公网IP:6443 \
--token="你的Token"
上面lb-server-port 默认的6443端口被其他应用占用了，所以我改成了5443，正常情况下不需要加~

更多的配置项目可以参考配置文档：

https://docs.rancher.cn/docs/k3s/installation/install-options/agent-config/_index/



没什么问题问题的，应该能看到新的节点上来了。












Ok的话，



去 vi /etc/systemd/system/k3s.service 改下，

然后重启k3s。

sudo systemctl daemon-reload
sudo systemctl restart k3s


基本完事。



一个完整的k3s.service 启动命令核心如下

ExecStart=/usr/local/bin/k3s \
    agent --node-name zj-hc2 \
    --lb-server-port 5443 \
    --node-ip 10.42.5.1  \
    -node-external-ip 外网IP，如果需要Ingress要写 \
    --server https://你的ServerIp:6443 \
    --token="你的Token"




到这里，我们的集群基本搞掂了。

简单测试了一下速度~










基本能跑得起来非敏感时延的服务了。

完事。



全文在飞驰的高铁上完成，还是值得一个赞的~

手动狗头~

祝大家假期快乐~