---
layout: post
title: 反手来个K8S入门到跑路
category: linux
date: 2019-06-09
tags:
- linux
- k8s
---
# 反手来个K8S入门到跑路

## 前言

放假前一两天发现腾讯云托管K8S集群上线好一阵子了, 还支持把原有主机迁入k8s集群, 索性开始搞事了.

先简单科普一下, 什么是k8s?

### k8s 科普时间

>  Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications. It groups containers that make up an application into logical units for easy management and discovery. Kubernetes builds upon 15 years of experience of running production workloads at Google, combined with best-of-breed ideas and practices from the community. 来自:[https://kubernetes.io/](https://kubernetes.io/)

```log
Kubernetes是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

Kubernetes一个核心的特点就是能够自主的管理容器来保证云平台中的容器按照用户的期望状态运行着（比如用户想让apache一直运行，用户不需要关心怎么去做，Kubernetes会自动去监控，然后去重启，新建.

总之，让apache一直提供服务），管理员可以加载一个微型服务，让规划器来找到合适的位置.

同时，Kubernetes也系统提升工具以及人性化方面，让用户能够方便的部署自己的应用（就像canary deployments）。

-- [https://www.kubernetes.org.cn/k8s](https://www.kubernetes.org.cn/k8s)

```

用人话来说, 首先我们以前已经把所有的服务(server/api)都Docker容器化了, 

k8s就是一套用来做容器服务管理/编排的系统.

那么我们现在开始吧.

## 神说, 要有k8s.

- 初始化自己的腾讯云/阿里云k8s 容器服务

登录一下自己的某云平台, 找一下"容器服务", 新建集群, 选"托管集群", 点点点之后完成配置.

然后把自己已有的机器加入到上面的集群里面去, 基本也是在界面上点点点就好.

### 简单科普一下k8s里面对于机器的定义

一个机器在k8s是一个节点,节点分成两种类型, 

- 一类是master节点, 跑着一堆k8s核心服务(etcd之类的, 具体的自己去了解吧);

- 一类是worker节点,用于跑我们自己的业务服务(如,某个爬虫)

一般情况下, 我们把服务扔给k8s运行起来的时候, 实际上都是在worker节点上起一个Docker容器实例 + 相关的配置.

这边云平台的托管集群就是master节点我们不管了, 云平台负责好, 需要的机器和配置都给他们搞;

我们把自己的已经买过的机器或者新买的机器加入到集群里面, 自己用就完事了.

所以....

我又来搞事了.

## 神说, 让我们要开始了

等下, 上面点完了, 先把本地的kubectl搞掂.

- 哈? 什么是kubectl. 

- 哦, 管理k8s的命令行工具.

- 没有图形界面吗?

- 有, 不过某云做的很难用, 原生的凑合能看, 便于理解概念的情况下, 还是先用ctl搞起吧.

安装教程: [通过 Kubectl 连接集群](https://cloud.tencent.com/document/product/457/8438)

<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

假装到这里应该都弄好了.

```sh
## 输入kubectl version 查看客户端和服务端版本信息, 能正常输出信息证明都work了
➜  codelover-blog git:(master) ✗ kubectl version
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.1", GitCommit:"b1b29978270dc22fecc592ac55d903350454310a", GitTreeState:"clean", BuildDate:"2018-07-18T11:37:06Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"12+", GitVersion:"v1.12.4-tke.1.1", GitCommit:"900490ac7b7ae6a13aa6599d43258f1164e9c458", GitTreeState:"clean", BuildDate:"2019-01-30T10:22:33Z", GoVersion:"go1.10.4", Compiler:"gc", Platform:"linux/amd64"}
```

### 又到概念时间 Pod/Service

- [Kubernetes Pod概述](http://docs.kubernetes.org.cn/312.html)

- [Kubernetes Services概述](https://www.kubernetes.org.cn/kubernetes-services)

用人话来说, 

Pod就是一堆app(一个/多个Docker容器实例)的集合;

Service就是在k8s内网坏境"对外"提供服务的一个实体;

我们直接在举个栗子来说明一下.

- 一个支持水平扩展的Web API服务, 我们为了高可用需要给它部署了三个实例, 这里就是一个Pod, 由三个实例组成

- "对外"(其他服务或者app需要访问它)来说, 我们需要直接访问它但是有没必要知道它有几个实例, 这个时候就有一个Service来承载它, Service负责把请求直接转发到Pod中, Pod里面有多少实例对Service和对调用者来说都是透明的.

所以我们来部署我们的第一套服务吧.

## 神说, 要开始入门了!

将下面的文件保存成sample.yaml, 然后执行 "kubectl apply -f sample.yaml"

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  # 给pod打上标签, 定义service 需要用
  labels:
    run: first-web-size
  name: first-web-size
spec:
  # 实例个数
  replicas: 2
  selector:
    matchLabels:
      run: first-web-size
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: first-web-size
    spec:
      containers:
      # 微软 dotnet core 样例镜像
      - image: mcr.microsoft.com/dotnet/core/samples:aspnetapp
        imagePullPolicy: IfNotPresent
        name: first-web-size
        terminationMessagePath: /dev/termination-log
        livenessProbe:
          httpGet:
            path: /v1/health
            port: 80
          initialDelaySeconds: 120
          periodSeconds: 30
        # CPU + 内存限制
        resources:
          # 最高使用500m的CPU, 超过就限制CPU;最高用512M内存, 超过就炸应用
          limits:
            cpu: 500m
            memory: 512Mi
          # 要求给50m的CPU调度和128M内存才可以启动
          requests:
            cpu: 50m
            memory: 128Mi
    # 这里是私有镜像仓库需要自己配置的一下秘钥什么之类的, 配置过程见k8s官方文档
    #   imagePullSecrets:
    #     - name: regcred
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: first-web-size
  annotations:
    # 这里:后面的值填腾讯云的子网id, 这样service就在腾讯云私有网络子网里面有自己的 ip了
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxx
spec:
  selector:
    # 过滤标签, 在上面定义pod的时候也写了对应的label
    run: first-web-size
  ports:
  - protocol: TCP
    # service对外提供访问的端口
    port: 80
    # pod的端口
    targetPort: 80
  type: LoadBalancer
  externalTrafficPolicy: Cluster
```

将上面的文件保存成sample.yaml, 然后执行 "kubectl apply -f sample.yaml", 可以看到下面的Log

```sh
➜  _posts git:(master) ✗ kubectl apply -f sample.yaml
deployment.extensions/first-web-size created
service/first-web-size created
➜  _posts git:(master) ✗ kubectl get pod
first-web-size-797fb6f8cc-q79df       1/1       Running     0          68s
first-web-size-797fb6f8cc-thwm7       1/1       Running     0          68s
➜  _posts git:(master) ✗ kubectl get service
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
first-web-size   LoadBalancer   10.3.255.248   <pending>     80:31054/TCP   101s

```

我们在上面可以看到, 有first-web-size这个Pod有两个实例, 有一个Service

这个时候登录和集群在同一个内网的服务器上面

执行 curl http://10.3.255.248  能看到对应的访问HTML
```sh
root@VM-0-6-ubuntu:~# curl http://10.3.255.248
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Home Page - aspnetapp</title>
```

怎么对外提供访问等我下次分享了.

用完了要删掉这些东西请执行 "kubectl delete -f sample.yaml"

拜...

我去做饭了.
