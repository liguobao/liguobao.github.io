---
title: 构建 meilisearch mini-dashboard 静态站点镜像
description: meilisearch, dashboard, static site
slug: meili-dashboard
date: 2025-01-16 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - 2025
  - meilisearch
  - linux
---

[mini-dashboard](https://github.com/meilisearch/mini-dashboard) 是 meilisearch 的一个小型查询看板，

提供了比较集群的查询页面，对于我的需求来说刚好够用。

不过官方的镜像构建只是个node Debug版本，

所以我自己构建了一个静态站点镜像。

```dockerfile
FROM node:20-alpine AS build-env

WORKDIR /home/node/app

RUN chown -R node:node /home/node/app

USER node

COPY package*.json ./

COPY --chown=node:node . .
RUN yarn install
ENV NODE_ENV=production
# if you want to use your own meilisearch server, you can set the following env
# ENV REACT_APP_MEILI_SERVER_ADDRESS=http://meilisearch:7700
# ENV REACT_APP_MEILI_API_KEY=masterKey
RUN yarn build

FROM nginx
COPY --from=build-env /home/node/app/build/ /usr/share/nginx/html/


```

## Docker with Nginx

nginx is also available in the Docker image. 

You can use it to serve the mini-dashboard.

```bash
docker build --build-arg REACT_APP_MEILI_SERVER_ADDRESS=http://meilisearch:7700 -t meilisearch-mini-dashboard-nginx . -f Dockerfile.nginx
docker run -p 8080:80 meilisearch-mini-dashboard-nginx
```

You can then access the mini-dashboard at `http://localhost:8080`.

大概如此。
