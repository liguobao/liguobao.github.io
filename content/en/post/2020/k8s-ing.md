---
layout: post
title: You Can Definitely Get Tencent Cloud K8s Service/Ingress Working
category: k8s
date: 2019-08-31
tags:
  - k8s
  - ingress
  - nginx
---

# Foreword

Previously we talked about Pods/Services. A Service gives you LB inside the cluster and a stable name for Pod‑to‑Pod access. But how do we expose it to the public internet?

In Kubernetes, Ingress handles external access (hosts, TLS, routing) — think “like nginx”, and many controllers are nginx‑based.

On Tencent Cloud, public IP LBs cost money (e.g., HTTP/HTTPS ALB ¥0.02/hour). If you can afford that, use the managed LB + Ingress flow in the console and click through.

## A frugal approach

Your worker nodes already have public IPs. Run nginx on a node and proxy to your K8s Service’s internal IP — done.

So how do we give a Service an internal IP reachable from the VPC? On Tencent Cloud, annotate the Service to allocate an internal LoadBalancer IP in your subnet.

Service example:

```yaml
apiVersion: v1
kind: Service
metadata:
  # Request an internal LB IP in your subnet
  annotations:
    service.kubernetes.io/loadbalance-id: lb-<your-lb-id>
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-<your-subnet-id>
  name: codelover-blog
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32278
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: codelover-blog
  sessionAffinity: None
  type: LoadBalancer
```

Check the result:

```text
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)
codelover-blog   LoadBalancer   172.16.255.232   172.17.0.28   80:32278/TCP

curl -v 172.17.0.28
... HTTP/1.1 200 OK ...
```

Now set up nginx on your public node:

```nginx
server {
  listen 80;
  server_name your.domain;
  location / { proxy_pass http://172.17.0.28:80; }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html { root html; }

  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

  gzip on; gzip_min_length 5k; gzip_buffers 4 16k; gzip_http_version 1.1;
  gzip_comp_level 3;
  gzip_types text/plain application/json application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
  gzip_vary on;
}
```

Done.

