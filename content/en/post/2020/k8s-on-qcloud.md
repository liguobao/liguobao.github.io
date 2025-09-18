---
layout: post
title: K8s From First Steps to Cleanup on Tencent Cloud
category: linux
date: 2019-06-09
tags:
- linux
- k8s
---
# Kubernetes on Tencent Cloud: From Zero to Running

## Intro

Tencent Cloud’s managed k8s had been around for a while and supports joining your own hosts. Time to experiment.

### What is k8s (quickly)?

Kubernetes is an open-source system for automating deployment, scaling, and management of containerized apps. It groups containers into logical units for easy management and discovery.

Plainly: we containerized our services already; k8s orchestrates and manages those containers.

## Step 1: Create a Cluster

- In your cloud console, find “Container Service”, create a managed cluster, click through the wizard.
- Join your existing or new instances as worker nodes.

Two node types:

- master: runs k8s control-plane (etcd, etc.)
- worker: runs your workloads (Pods)

Managed clusters hide master ops; you manage/join your workers.

## Step 2: kubectl

Install kubectl and connect to the cluster (see your cloud docs). Verify:

```sh
kubectl version
```

## Concepts: Pod and Service

- Pod: one or more container instances of an app.
- Service: stable, virtual IP/endpoint inside the cluster that load-balances to Pod replicas.

Example: A scalable web API runs 3 replicas in one Deployment/Pod template; a Service exposes a single endpoint that forwards to replicas.

## First Deployment & Service

Save as `sample.yaml`, then run `kubectl apply -f sample.yaml`.

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: first-web-size
  name: first-web-size
spec:
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
      labels:
        run: first-web-size
    spec:
      containers:
      - image: mcr.microsoft.com/dotnet/core/samples:aspnetapp
        imagePullPolicy: IfNotPresent
        name: first-web-size
        livenessProbe:
          httpGet:
            path: /v1/health
            port: 80
          initialDelaySeconds: 120
          periodSeconds: 30
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 50m
            memory: 128Mi
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: first-web-size
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxx
spec:
  selector:
    run: first-web-size
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
  externalTrafficPolicy: Cluster
```

Apply and check:

```sh
kubectl apply -f sample.yaml
kubectl get pods
kubectl get service first-web-size
```

From a host on the same VPC:

```sh
curl http://<CLUSTER-IP>
```

Cleanup:

```sh
kubectl delete -f sample.yaml
```

