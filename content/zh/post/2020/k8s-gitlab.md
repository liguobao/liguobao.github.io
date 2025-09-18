---
layout: post
title: 客官来玩K8S之搭个Gitlab
category: git
date: 2019-08-31
tags:
  - gitlab
  - CI/CD
---

# 客官来玩K8S之搭个Gitlab

## 前言

不需要前言.

## 本次教程中包含的知识点

需要的知识点

- k8s基础, Pod/Service 相关知识

- k8s存储, PV/PVC/StorageClass 相关知识

- nginx基础

### Gitlab CE 依赖的服务

- postgreSQL

- redis

- 2G 以上内存, 最好 4G(2G 内存不如 dog)

当然, 以上服务一样用 K8S 部署.

#### postgreSQL

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: postgresql
  name: postgresql
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: postgresql
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: postgresql
    spec:
      containers:
        - env:
          - name: POSTGRES_DB
            value: gitlab
          - name: POSTGRES_USER
            value: gitlab
          - name: POSTGRESQL_PASSWORD
            value: 随便写你的密码
          - name: PGDATA
            value: /var/lib/postgresql/data/pgdata
          image: postgres:10
          imagePullPolicy: IfNotPresent
          name: postgresql
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 5432
            protocol: TCP
          resources:
            limits:
              cpu: 500m
              memory: 1024Mi
            requests:
              cpu: 200m
              memory: 256Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: /var/lib/postgresql/data/
            name: es-data
            subPath: postgresql
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      # 这里是k8s的pvc, 我这里pvc叫es-data, 和elastic用同一个磁盘
      volumes:
        - name: es-data
          persistentVolumeClaim:
            claimName: es-data
---
apiVersion: v1
kind: Service
metadata:
  # 这个annotations是腾讯云申请内网IP的配置, 需要改成自己k8s所在网络的子网id
  # postgresql 这里其实可以不要内网service ip, 内网直接用service name访问即可
  annotations:
    service.kubernetes.io/loadbalance-id: lb-你自己的loadbalanceid
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-你自己的子网id
  labels:
    run: postgresql
  name: postgresql
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 3433
    protocol: TCP
    targetPort: 5432
  selector:
   run: postgresql
  sessionAffinity: None
  type: LoadBalancer
```

这里面用的 PV/PVC 需要自己在腾讯云里面创建, 基本就是点点点就能创建出来了.

#### redis 部署

跳过...

随便抄一下 k8s 部署 Redis 教程就完事了.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  generation: 2
  labels:
    app: redis
  name: redis
spec:
  podManagementPolicy: OrderedReady
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: redis
  serviceName: redis
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: redis
    spec:
      containers:
        - env:
            - name: REDIS_PASSWORD
              value: 换成你的密码
          image: bitnami/redis:5.0
          imagePullPolicy: IfNotPresent
          name: redis
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 200m
              memory: 128Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /bitnami/redis/data
              name: redis-data
              subPath: redis
      dnsPolicy: ClusterFirst
      imagePullSecrets:
        - name: regsecret
      nodeSelector:
        tuiwen-tech.com/phase: test
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        runAsUser: 0
      terminationGracePeriodSeconds: 30
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  volumeClaimTemplates:
    - metadata:
        creationTimestamp: null
        name: redis-data
      spec:
        accessModes:
          - ReadWriteOnce
        dataSource: null
        resources:
          requests:
            storage: 10Gi
        storageClassName: 换成你的storageClassName
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis
  name: redis
spec:
  externalTrafficPolicy: Cluster
  ports:
    - name: headless
      nodePort: 31966
      port: 6379
      protocol: TCP
      targetPort: 6379
  selector:
    app: redis
  sessionAffinity: None
  type: LoadBalancer
```

### Gitlab CE

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: gitlab
  name: gitlab
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
        - env:
            - name: GITLAB_OMNIBUS_CONFIG
              value: |
                # external_url 这里要注意, 要不改成你的公网IP, 要不改成nginx暴露到外面的域名+端口 或者域名+二级目录
                external_url '换成你自己的'
                # Disable the built-in Postgres
                postgresql['enable'] = false

                # Fill in the connection details for database.yml
                gitlab_rails['db_adapter'] = 'postgresql'
                gitlab_rails['db_encoding'] = 'utf8'
                gitlab_rails['db_host'] = 'postgresql'
                gitlab_rails['db_port'] = 3433
                gitlab_rails['db_database'] = "gitlab"
                gitlab_rails['db_username'] = 'gitlab'
                gitlab_rails['db_password'] = '换成你自己的'

                # mail config
                gitlab_rails['smtp_enable'] = true
                gitlab_rails['smtp_address'] = "换成你自己的"
                gitlab_rails['smtp_port'] = 465
                gitlab_rails['smtp_user_name'] = "换成你自己的"
                gitlab_rails['smtp_password'] = "换成你自己的"
                gitlab_rails['smtp_domain'] = "换成你自己的"
                gitlab_rails['smtp_authentication'] = "login"
                gitlab_rails['smtp_enable_starttls_auto'] = true
                gitlab_rails['smtp_tls'] = true  # 这个很重要，而且是官方文档里没提及的 

                # If your SMTP server does not like the default 'From: [email protected]ocalhost' you
                # # can change the 'From' with this setting.
                gitlab_rails['gitlab_email_from'] = '完整邮件账户'

                # Disable the built-in Redis
                redis['enable'] = true

                # Add any other gitlab.rb configuration here, each on its own line
                # Redis via TCP
                gitlab_rails['redis_host'] = 'redis'
                gitlab_rails['redis_port'] = 6379
                gitlab_rails['redis_password'] = '换成你自己的'
          image: gitlab/gitlab-ce:11.11.7-ce.0
          name: gitlab
          resources:
            requests:
              memory: "1Gi"
              cpu: "300m"
            limits:
              memory: "3Gi"
              cpu: "1000m"
          ports:
            - containerPort: 443
            - containerPort: 80
            - containerPort: 22
          volumeMounts:
            - mountPath: /etc/gitlab
              name: es-data
              subPath: gitlab
            - mountPath: /var/opt/gitlab/git-data
              name: es-data
              subPath: gitlab
      restartPolicy: Always
      serviceAccount: gitlab
      serviceAccountName: gitlab
      volumes:
        - name: es-data
          persistentVolumeClaim:
            claimName: es-data
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/loadbalance-id: 换成你自己的
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: 换成你自己的
  labels:
    app: gitlab
  name: gitlab
spec:
  ports:
    - name: git-ssl
      port: 443
      targetPort: 443
    - name: git-http
      port: 80
      targetPort: 80
  selector:
    app: gitlab
  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/loadbalance-id: 换成你自己的
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: 换成你自己的
  labels:
    app: gitlab
  name: gitlab-ssh
spec:
  ports:
    - name: git-ssh
      port: 22
      targetPort: 22
  selector:
    app: gitlab
  type: LoadBalancer
```

理论上来说, 只需要等待启动就完事了..

### 最后是暴露到外部的 NGINX 配置

如果直接使用 k8s ingress 拿到公网 IP 的话, 就不用自己配置 NGINX 转发了.

如果和我一样需要自己用 Nginx 提供外部访问的话, 参考下面.

```nginx

location /app/git {
    proxy_pass   http://可达的内网IP(就是k8s service的内网IP):80;
    client_max_body_size 50m;
}

```

教程完毕.
