---
title: 宇树机器人G1便携开发指北：基于树莓派的开发环境搭建
description: 本文档介绍了如何在树莓派上进行宇树机器人G1的便携开发，涵盖环境配置、代码获取及基本操作步骤。
slug: unitree-g1-develop-on-raspberry-pi
date: 2026-01-24 08:00:00+0000
# image: girl.png
categories:
  - linux
tags:
  - unitree
  - raspberry-pi
  - robotics
  - development
---

# 宇树机器人G1便携开发指北：基于树莓派的开发环境搭建

## 引言

如果要调试 unitree-g1-edu 机器人的话，肯定是看过他们的官方文档的。

快速开发文档在这里：[G1_developer/快速开发](https://support.unitree.com/home/zh/G1_developer/quick_development)

环境要求倒没撒，基于Ubuntu 20.04 or 22.04, 装上依赖库，

基本是有手就能行的操作。

只是，如果你想去调试一下G1机器人的话，

官方的方案是，大概是这样的。


[演示视频](https://oss-global-cdn.unitree.com/static/0818dcf7a6874b92997354d628adcacd.mp4)。

- 拖一根网线从G1机器人颈部网口接入，连接到你的Ubuntu主机，
- 给Ubuntu主机网络设定固定IP, 通过lan口做网络通讯

大概如下：

![](/img/post/unitree-g1-develop-on-raspberry-pi/1.lan.png)

```text
建议新用户使用网线与转接线将用户计算机接入 G1 交换机，

并将与机器人通信的网卡设置在 192.168.123.X 网段下，

推荐使用 192.168.123.99。

```

倒没什么问题，只是拖着一个网线，怎么看都是麻烦。

有什么别的办法么？

思索了一下，直接用“树莓派”做个主控机器连接G1网络，

同时树莓派连接到本地局域网，那不就可以直接远程是操控机器了？

又研究了一下，G1-EDU网卡旁边还有三个供电口，分别是12V 、 24V 、 48V，

也足够给树莓派供电了（树莓派是5V，买个电压转换器即可）。


## “树莓派” 硬件环境准备

转压器 + 电源线，加起来三十来块搞掂，

淘宝直接买就可以。

3M胶布固定“树莓派”到机器背面，电源线接上去。

