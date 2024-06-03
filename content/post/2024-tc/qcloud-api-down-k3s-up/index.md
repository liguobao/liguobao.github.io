---
title: 腾讯云API网关废了？集群开源方案平替
description: 腾讯云,API网关,k8s,k3s,ingress
slug: q-cloud-api-down-k3s-up
date: 2024-06-03 08:00:00+0000
image: qcloud.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---

![](./api-gateway.png)

## 背景

听过腾讯云API网关要下线了，真的太难受了，之前的项目都（bu）是（是）用的这个，现在要换成什么呢？

[【重要】API 网关产品停止售卖公告](https://cloud.tencent.com/document/product/628/106651)

然而并不是，这玩意当年我还在白嫖腾讯云k8s master集群的时候，

某一次升级之后他们把k8s的service 切成了API网关，然后开始要求我按量付费。

接着我就开始研究怎么把这个东西给替换掉~

## 方案一：腾讯云集群 + nodePort

- 集群还是一样的集群，自己的机器作为worker节点接入集群
- 直接删掉所有带内网IP的service，全部使用nodePort暴露服务。
- 在worker节点上安装NGINX，使用NGINX的反向代理功能，将请求转发到nodePort上。


这个方案跑了大概一年多左右，集群也开始收费了，美名曰管理费~

算下来一个月几十块，一年几百块，并不能接受这个价格。

遂，方案迁移~

## 方案二：SuperEdge 自建集群 + nodePort + nginx

自建集群嘛，什么方便什么来。

所以当年还写过这些文章：


[知乎专栏：边缘计算k8s集群SuperEdge初体验](https://zhuanlan.zhihu.com/p/346084530)

[知乎专栏：SuperEdge边缘计算集群挂载NFS](https://zhuanlan.zhihu.com/p/514678542)

妥妥算是他们的一个自来水了。

然而...

大概是在去年的时候，SuperEdge 微信群老群说迁移，新群加进去没人管了。

私聊了一下之前熟悉的技术，说项目进入维护状态了。

![](./super-edge-bug-report.png)

啧啧啧...

问题是，这玩意证书过期没法renew啊，我的集群直接崩了。

真心累了，国内的大厂的开源项目，大多逃不过KPI的命运。

玩不起不好进场，玩不转就只能被淘汰。

真的，再信国内的大厂开源项目，我就是猪头！！!

接着开始考察新的方案： 

- 依旧是All In K8s
- 依旧是自建集群
- 依旧是不花一分钱


然后，发现 k3s 这个东西，真香~

## 最终方案：k3s + ingress-nginx

部署方案见：[知乎专栏：【k3s】年度最佳边缘计算集群方案](https://zhuanlan.zhihu.com/p/696737105)

node节点大概如下：

```sh
NAME              STATUS                     ROLES                       AGE     VERSION
aliyun-bj-199     Ready                      <none>                      34d     v1.29.4+k3s1
haru              Ready                      <none>                      6d16h   v1.29.4+k3s1
t7610             Ready                      <none>                      6d2h    v1.29.4+k3s1
vm-16-12-ubuntu   Ready                      control-plane,etcd,master   75d     v1.28.7+k3s1
vm-28-17-ubuntu   Ready,SchedulingDisabled   control-plane,etcd,master   75d     v1.28.7+k3s1
zj-hc1            Ready                      <none>                      33d     v1.29.4+k3s1
```

在上面其实可以看到，master 节点就两台，vm-16-12-ubuntu 和 vm-28-17-ubuntu ，

两台机器在云端，内网互通，都有公网IP。（划重点）

其他的机器部分在本地，部分在其他云，部分在边缘设备上，内网互通，但是不一定有公网IP。

照着以前的逻辑，我们是需要一个API 网关，把流量转发到到worker节点上。

这也是现在某云的API网关的功能，但是我们现在没有了。

不过，没有了API网关，我们还有 ingress-nginx。

k3s 集群中，有公网IP的master节点，我们可以直接在上面部署 ingress-nginx。

直接作为流量入口。