---
layout: post
title: 没有执行过rm -rf /*的开发不是好运维
category: linux
date: 2018-11-20
tags:
- linux
- rm
---

# 没有执行过rm -rf /*的开发不是好运维

## 起因

突然收到用户反馈说网站在手机端打开是白屏, 很奇怪的问题.

在电脑端试了下,确实也是白屏,HTML加载进来了,好像有个核心JS加载失败.

看到一个错误是: We're sorry but house doesn't work properly without JavaScript enabled. Please enable it to continue.

还有一个http请求的错误是: ERR_INCOMPLETE_CHUNKED_ENCODING

![](https://ws1.sinaimg.cn/large/64d1e863gy1fxk2pfnj1tj22lc10igrm.jpg)

于是尝试了一下的解决方案:

- 无脑重启看看能不能解决问题?重启了一下对应Docker容器,无果

- 可能是现在版本引入的Bug?回滚代码重新build,无果.

- nginx的问题? 重启nginx,无果.

- 查看nginx日志,没什么有用的东西,无果.

灵机一闪,不会是磁盘空间满了吧. 

df -h 看了一眼,99.99%的磁盘使用率.

某个Docker容器的磁盘空间用掉了34G.

看一眼Docker容器,直觉告诉我应该是Elasticsearch服务...

不算太重要的服务,先停了清理空间再说.

删掉了容器删了data文件,重启nginx,一切都正常Work了.

问题解决!!!

不过Elasticsearch总要重新回复回来嘛,看了下腾讯云云硬盘盘价格,也不是很贵嘛.

单独给Elasticsearch 起个数据盘吧.

## 作死开始

首先根据腾讯云的指示,挂在了数据盘到服务器上面.

然后给数据盘分区,接着mount到对应的路径.


嗯,好像有个警告.

难道不是这个磁盘么?换另外一个看看.

执行另外一个mount.

全程命令如下:

![](https://ws1.sinaimg.cn/large/64d1e863gy1fxk3f86li1j220613mkho.jpg)

进入对应目录清空一下云盘数据吧.(PS:脑子有病才做这个,刚刚初始化的云盘哪有东西.)

ls 看一下,咋这么多奇奇怪怪的文件,难道是原来Elasticsearch docker 容器留下来的.

先删了再说.

执行 rm -rf ./*

咦,怎么有文件busy无法删除.

额,咋ls都没有了.

哈?cat 也没有了.

噗,copy也炸了.

cd 还在.

哇卡,这可咋办了.

## 先复盘一下做了什么事情

- 初始化磁盘的时候没有格式化,但是mount失败

- mount失败后没有检查原因,直接尝试把另一个磁盘mount进去

- mount系统盘到指定文件夹后并没有检查内容,直接rm -rf ./*

- rm -rf ./* 此时已经基本没救了

## 拯救尝试

还在跑的服务基本是活着的，所以暂时来说API和Web网站都是好的。

服务器上面跑的基本都是Docker容器, Docker镜像都在阿里云上面存着，基本不怕丢失的问题。

不过应用配置文件/服务器证书之类的东西都在上面，这个估计要折腾一下了。

cd 还能用，ls没了，cat也没了。

尝试cat xxx.conf也没用了,难道只能一点点翻配置文件么.

群里的朋友提了一句，看看你的云盘有没有备份之类的.

咦,好像两个星期前找腾讯云技术支持的时候做过一次系统镜像.

是不是可以直接拿回来直接用?

看了下具体的镜像版本和备注信息,看起来那时候上面的内容和现在的估计没太多变化.

直接重装之后更新一下各个服务的镜像到最新版本应该就好了.

## 放弃拯救,直接使用备份的系统镜像重装

Work...

系统备份镜像拯救世界!!!

## 后续操作 + 总结

- 数据盘和系统盘分开,不要让程序的数据导致系统不可用

- 在费用允许的情况下设置磁盘快照策略,我这边最极端的情况下也应该能回滚到一个星期前的版本

- 下次大影响操作前先手动备份系统镜像,救命稻草一般的存在.

