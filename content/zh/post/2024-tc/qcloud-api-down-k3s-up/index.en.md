---
title: Tencent Cloud API Gateway Sunset? Open-Source Cluster Replacement
description: Tencent Cloud, API Gateway, k8s, k3s, ingress
slug: q-cloud-api-down-k3s-up
date: 2024-06-03 08:00:00+0000
image: qcloud.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---

## Background: API Gateway Is Going Away

![](/pics/api-gateway.png)

Heard Tencent Cloud’s API Gateway is being sunset. Some of our projects were (unfortunately) built on it. What to replace it with now?

[Important: API Gateway product end-of-sale announcement](https://cloud.tencent.com/document/product/628/106651)

Back when I was still using Tencent Cloud’s managed k8s masters for free, after one upgrade they switched our Services to API Gateway behind the scenes and started billing by usage.

So I began looking into how to remove and replace it — and today that work finally pays off.

## Option 1: Tencent Cloud k8s + NodePort

- Keep the same cluster; join your own machines as worker nodes.
- Remove all Services bound to internal IPs; expose everything via NodePort.
- Install NGINX on worker nodes and reverse-proxy traffic to NodePort.

This ran for about a year. Then the managed cluster itself started charging a “management fee”. A few dozen RMB per month, hundreds per year — not acceptable for us. Time to migrate again.

## Option 2: SuperEdge self-hosted cluster + NodePort + NGINX

For a self-hosted cluster, use whatever is convenient.

I wrote these back then:

- SuperEdge first impressions (Chinese): https://zhuanlan.zhihu.com/p/346084530
- SuperEdge + NFS (Chinese): https://zhuanlan.zhihu.com/p/514678542

I was basically an organic promoter for the project.

However… last year the old SuperEdge WeChat group was migrated; the new group seemed unmaintained. People I knew said the project moved into maintenance mode.

![](/pics/super-edge-bug-report.png)

The real problem: the certs expired and wouldn’t renew; my cluster broke.

Honestly, it’s tiring — many big-company open-source projects in CN end up tied to KPIs. Hard to get in, harder to keep running, and easy to get dropped.

So I started looking again, with these goals:

- Still all‑in on k8s
- Still self‑hosted
- Still zero cost

Then I found k3s — it’s great.

## Final Approach: k3s + ingress-nginx

Deployment guide: https://zhuanlan.zhihu.com/p/696737105 (Chinese)

Example node list:

```sh
NAME              STATUS                     ROLES                       AGE     VERSION
aliyun-bj-199     Ready                      <none>                      34d     v1.29.4+k3s1
haru              Ready                      <none>                      6d16h   v1.29.4+k3s1
t7610             Ready                      <none>                      6d2h    v1.29.4+k3s1
vm-16-12-ubuntu   Ready                      control-plane,etcd,master   75d     v1.28.7+k3s1
vm-28-17-ubuntu   Ready,SchedulingDisabled   control-plane,etcd,master   75d     v1.28.7+k3s1
zj-hc1            Ready                      <none>                      33d     v1.29.4+k3s1
```

As you can see, there are two masters (vm-16-12-ubuntu and vm-28-17-ubuntu). They’re on cloud VMs, mutually reachable on the private network, and both have public IPs (important).

Other nodes are a mix of local, other clouds, and edge devices. They can talk over private network, but not all have public IPs.

Previously, we needed an API Gateway to forward traffic to worker nodes. That’s what Tencent Cloud’s API Gateway did, but we don’t have it anymore. Instead, we can use ingress-nginx.

In a k3s cluster, any node with a public IP can serve as an entry point.

We can configure ingress-nginx to forward traffic to services living on worker nodes.

All you need is to pass `--node-external-ip` when starting the server/agent to set the public IP.

```sh
# server
ExecStart=/usr/local/bin/k3s \
    server \
	--node-external-ip PUBLIC_IP \
    --flannel-backend wireguard-native \
	'--tls-san' \
	'PUBLIC_IP' \
	'--node-name' \
	'vm-16-12-ubuntu' \
	'--server' \
	'https://10.0.28.17:6443' \

# agent
ExecStart=/usr/local/bin/k3s \
    agent --node-name zj-hc1 \
    --lb-server-port 5443 \
    --node-ip 10.42.4.1  \
    -node-external-ip PUBLIC_IP \
    --server https://server:6443 \
    --token="YOUR_TOKEN" \
```

Now ports 80/443 on that node can accept external traffic (make sure your firewall allows it). Point your domain’s DNS A record(s) to that node’s public IP, and you’re good to go.

## A Minimal Working Example

- Self-signed or ACME certs supported
- Any node with a public IP can act as an entry point
- For LB, add multiple A records or use DNS load balancing

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: stirling-pdf
spec:
  capacity:
    storage: 30Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/cfs-a4nopkhh/stirling-pdf
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - zj-hc1
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: stirling-pdf
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
  storageClassName: local-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stirling-pdf
  labels:
    app: stirling-pdf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stirling-pdf
  template:
    metadata:
      labels:
        app: stirling-pdf
    spec:
      containers:
        - name: stirling-pdf
          image: ccr.ccs.tencentyun.com/liguobao/s-pdf:ustc
          ports:
            - containerPort: 8080
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
              port: 8080
            initialDelaySeconds: 120
            periodSeconds: 60
          env:
            - name: DOCKER_ENABLE_SECURITY
              value: "false"
            - name: INSTALL_BOOK_AND_ADVANCED_HTML_OPS
              value: "false"
            - name: UI_APP_NAME
              value: "R2049 PDF"
            - name: ALLOW_GOOGLE_VISIBILITY
              value: "true"
            - name: LANGS
              value: "zh_CN"
            - name: APP_LOCALE
              value: "zh_CN"
          volumeMounts:
            - mountPath: /configs
              subPath: configs
              name: data
            - mountPath: /usr/share/tessdata
              subPath: tessdata
              name: data
            - mountPath: /customFiles/
              subPath: customFiles
              name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: stirling-pdf
      imagePullSecrets:
        - name: regcred
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: stirling-pdf-service
spec:
  selector:
    app: stirling-pdf
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
  generation: 1
  name: ingress-pdf-house2048
spec:
  ingressClassName: traefik
  rules:
    - host: pdf.house2048.cn
      http:
        paths:
          - backend:
              service:
                name: stirling-pdf-service
                port:
                  number: 8080
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - pdf.house2048.cn
      secretName: pdf-house2048-cn-tls
```

