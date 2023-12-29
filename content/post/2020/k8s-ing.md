---
layout: post
title: 你不可能搞不定腾讯云K8S的Service/Ingress
category: k8s
date: 2019-08-31
tags:
  - k8s
  - ingress
  - nginx
---

# 你不可能搞不定腾讯云K8S的Service/Ingress

## 前言

上一个手把手教程上简单介绍过k8s中的Pod/Service, 

里面提到过Service是K8S中的负载均衡, 同时也可以在不同Pod中使用Service name互相访问.

如果我们在外部(公网)访问K8S的服务怎么办呢? 

首先, 你的有个公网IP.

然后....关掉这个教程.


回到正文.

在k8s中, ingress来负责对外提供访问, 一般域名配置, 域名证书都在这边.

是不是有点熟悉? 这不就是NGINX? 

对的, 甚至ingress的实现就有一部分是NGINX.

但是, 大家都知道, 公网IP这种东西, 可是要收费的.

	
腾讯云的收费标准: 应用型负载均衡器（支持HTTP/HTTPS）0.02元/小时

有钱的同学直接用就好了, 

教程见腾讯云集群管理, 点点点就好了.

## 穷人不配用公网? 不, 我们有自己的倔强!

想下, 我们在做k8s集群的时候, 用的主机不是本来就带公网IP吗?

虽然现在主机已经是k8s的worker节点了, 不过机器还是可以继续用的呀.

所以, 我们的方案是, 

主机上装个NGINX, 由NGINX用端口转发的方式转到内部K8S Service中 ,

这样就完美了.

这样的话, 那么我们的主机怎么访问k8s Service呢? 

在前面我们简单提过, k8s的pod,service都是在自己的内网的.

虽然现在主机和k8s的服务可能都在一个主机上, 然而里面还是有逻辑层内网隔离的.

那么, 这样怎么办呢? 

放心, k8s service早就有解决方案啦.

答案是:

在腾讯云中的使用service.kubernetes.io/qcloud-loadbalancer-internal-subnetid annotations来申请内网IP.

```yaml
---

apiVersion: v1
kind: Service
metadata:
  # 这个annotations是腾讯云申请内网IP的配置, 需要改成自己k8s所在网络的子网id
  # postgresql 这里其实可以不要内网service ip, 内网直接用service name访问即可
  annotations:
    service.kubernetes.io/loadbalance-id: lb-你自己的loadbalanceid
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-你自己的子网id
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


查看一下配置的效果

```log
➜  codelover-blog git:(master) ✗ kc get service                  
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
codelover-blog   LoadBalancer   172.16.255.232   172.17.0.28   80:32278/TCP                 55d


root@VM-0-13-ubuntu:~# curl -vvvv 172.17.0.28
* Rebuilt URL to: 172.17.0.28/
*   Trying 172.17.0.28...
* Connected to 172.17.0.28 (172.17.0.28) port 80 (#0)
> GET / HTTP/1.1
> Host: 172.17.0.28
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.17.1
< Date: Sun, 01 Sep 2019 05:30:51 GMT
< Content-Type: text/html
< Content-Length: 45988
< Last-Modified: Mon, 08 Jul 2019 03:59:21 GMT
< Connection: keep-alive
< ETag: "5d22bf99-b3a4"
< Accept-Ranges: bytes
<
<!DOCTYPE html>


```

现在内网IP有了, 下面也就是配置一下NGINX的事情了.

```nginx
    server {
            listen 80;
            server_name  你的域名;
            location / {
                proxy_pass   http://172.17.0.28:80;
            }

            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                    root   html;
            }
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	          gzip on;
            gzip_min_length 5k;
            gzip_buffers 4 16k;
            gzip_http_version 1.1;
            gzip_comp_level 3;
            gzip_types text/plain application/json application/javascript text/css application/xml text/javascript image/jpeg image/gif image/png;
            gzip_vary on;
    }
```

好了, 完事.

