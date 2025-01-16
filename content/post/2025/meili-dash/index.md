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
FROM node:20-alpine as build-env

WORKDIR /home/node/app

RUN chown -R node:node /home/node/app

USER node

COPY package*.json ./

COPY --chown=node:node . .
RUN yarn install
ENV NODE_ENV=production
# if you want to use your own meilisearch server, you can set the following env
# ENV REACT_APP_MEILI_SERVER_ADDRESS=http://you-server:7700
# ENV REACT_APP_MEILI_API_KEY=masterKey
RUN yarn build

FROM nginx
COPY --from=build /home/node/app/build/ /usr/share/nginx/html/
expose 80

```

大概如此。

