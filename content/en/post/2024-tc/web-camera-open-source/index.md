---
title: WebCamera — A WebRTC-Based Peer-to-Peer Webcam
description: WebRTC, open-source project
slug: open-source-web-camera
date: 2024-06-12 08:00:00+0000
image: web-c.png
categories:
  - TC
tags:
  - Github
  - Open Source
---

## WebCamera — A WebRTC-Based Peer-to-Peer Webcam

Saw a neat open-source project on Twitter/X — a P2P webcam built on WebRTC.

- Repo: https://github.com/ShouChenICU/WebCamera

Looked through the code and docs — interesting enough to deploy and try. Sharing notes below.

## Project Overview

- WebCamera is a network webcam tool built on WebRTC, developed with Nuxt.js and managed with Yarn.

### Features

- Real-time video: WebRTC-based, efficient streams.
- Cross-platform: Works across browsers and devices.
- Dev-friendly: Built on Nuxt.js, easy to extend and maintain.
- Modular design: Easy to integrate and expand.
- Privacy & security: Peer-to-peer encrypted connections.

## Docker Deployment

- Image: ccr.ccs.tencentyun.com/liguobao/tools:web-camera
- This is my built image hosted on Tencent Cloud Container Registry.

Build it yourself:

```sh
docker build -t webcamera .
docker run -d -p 3000:3000 webcamera
```

### k8s Deployment

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

## What’s Next

Chatted with the author — there might be a simple online meeting app coming next, worth keeping an eye on.

## References

- WebCamera: https://github.com/ShouChenICU/WebCamera
- WebRTC: https://webrtc.org/?hl=zh-cn
- Ruanyifeng’s WebRTC docs (Chinese): https://javascript.ruanyifeng.com/htmlapi/webrtc.html

WebRTC is quite interesting — worth exploring if you’re curious. The author said they only studied it for a couple of days before shipping this.

