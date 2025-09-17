---
title: k3s Daily ImagePullError Memo
description: k3s image pull error
slug: k3s-image-pull-error
date: 2025-06-07 08:00:00+0000
categories:
  - TC
tags:
  - 2025
  - k3s
  - docker
  - linux
---

When doing things in China, you have to solve network issues daily.

It's okay locally, but it's a bit painful on the server.

Today when fiddling with the k3s cluster, I accidentally wiped out etcd.

So I started reinstalling the entire cluster.

Everything else is fine, the images are all on some cloud, but traefik is painful.

This thing uses docker hub source.

If it was a few years ago, it would be fine, there were so many mirror sources in China, just configure one and it's done.

But after last year, the mirror sources died one by one.

Can't give the server a scientific internet either.

And the premise of scientific internet is that you have to scientific internet first to download things.

Dead end.

After fiddling for several hours, I had a sudden idea.

Just export the image locally and import it, isn't that done.

k3s although it doesn't use docker runtime, but it has compatible commands.

Asked GPT, very simple.

```bash
docker save docker.io/traefik:v3.4.1 -o traefik_v3.4.1.tar

scp traefik_v3.4.1.tar root@my-ip:~/

ctr -n k8s.io images import /root/traefik_v3.4.1.tar
```

Done.

Yes. That's how simple it is.

Everything is so harmonious.
