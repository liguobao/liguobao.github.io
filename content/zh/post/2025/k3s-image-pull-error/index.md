---
title: k3s日常ImagePullError备忘录
description: k3s image pull error
slug: k3s-image-pull-error
date: 2025-06-07 08:00:00+0000
categories:
  - TC
tags:
  - 2025
  - k3s
  - docker
  - linux
---


在国内搞事情嘛，日常得解决网络问题。

在本地倒好说，在服务器就有点蛋疼了。

今天折腾k3s集群的时候，不小心把etcd干没了。

于是开始重装整个集群。

别的都好说，反正镜像都在某云，然鹅traefik就蛋疼了。

这玩意用的是docker hub源。

如果是早些年也好说，国内那么多的镜像源，随便配一个也就完事了

但是去年之后，镜像源死得七七八八了。

总不能给服务器也挂个科学上网吧。

而且科学上网的前提是你还得先科学上网出去下载东西。

死局。





折腾了几个小时之后突发奇想。

直接在本地导出镜像再导入不就完事了。

k3s 虽然用的不是docker运行时，但也有兼容的命令啊。

问了下GPT，很简单。

```bash

docker save docker.io/traefik:v3.4.1 -o traefik_v3.4.1.tar

scp traefik_v3.4.1.tar root@my-ip:~/

ctr -n k8s.io images import /root/traefik_v3.4.1.tar

```

完事了。



对。就这么简单完事了。


一切都那么和谐。

