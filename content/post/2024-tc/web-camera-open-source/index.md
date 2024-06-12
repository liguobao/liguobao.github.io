---
title: WebCamera: 基于WebRTC的点对点网络摄像头
description: WebRTC,开源项目
slug: web-camera-open-source
date: 2024-06-12 08:00:00+0000
image: web-c.png
categories:
  - TC
tags:
  - zhihu
  - open-source
  - 2024
---

# WebCamera: 基于WebRTC的点对点网络摄像头

- [github.com/ShouChenICU/WebCamera](https://github.com/ShouChenICU/WebCamera)

前几天逛推，看到有个大佬开源了个小项目，基于WebRTC的点对点网络摄像头。

顺手看了一眼代码和文档，发现这个项目还是挺有意思的，于是自己部署了一个，分享给大家。

## 项目介绍

- WebCamera 是一个基于 WebRTC 技术的网络摄像头工具站，使用 Nuxt.js 框架开发，并通过 Yarn 进行包管理。

### 特性

- 实时视频流: 使用 WebRTC 技术实现高效的实时视频流。
- 跨平台支持: 兼容多种浏览器和设备。
- 易于开发: 基于 Nuxt.js 框架，方便扩展和维护。
- 模块化设计: 便于功能的扩展和集成。
- 隐私安全: 使用点对点加密连接，保护隐私安全。

## docker部署

- ccr.ccs.tencentyun.com/liguobao/tools:web-camera
- 是我构建的镜像，托管在腾讯云容器镜像服务上。


自行构建如下：

```sh
docker build -t webcamera .
docker run -d -p 3000:3000 webcamera

```

### k8s 部署

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-camera
  labels:
    app: web-camera
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-camera
  template:
    metadata:
      labels:
        app: web-camera
    spec:
      containers:
        - name: web-camera
          image: ccr.ccs.tencentyun.com/liguobao/tools:web-camera
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2048Mi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 120
            periodSeconds: 60
          env:
            - name: DOCKER_ENABLE_SECURITY
              value: "false"
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: web-camera
spec:
  selector:
    app: web-camera
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
  generation: 1
  name: ingress-web-camera
spec:
  ingressClassName: traefik
  rules:
    - host: xxx.yyy.cn
      http:
        paths:
          - backend:
              service:
                name: web-camera
                port:
                  number: 3000
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - xxx.yyy.cn
      secretName: xxx-yyy-cn-tls
```

## 后续

和大佬聊了下，后续可能会做个简单的在线视频会议的项目，可以期待一下~

## 参考

- [WebCamera](https://github.com/ShouChenICU/WebCamera)
- [WebRTC](https://webrtc.org/?hl=zh-cn)
- [阮一峰：WebRTC 文档](https://javascript.ruanyifeng.com/htmlapi/webrtc.html)

WebRTC 这个协议有点意思，有兴趣的朋友可以多了解一下~

毕竟大佬都说也就看了两天，就写出来了~
