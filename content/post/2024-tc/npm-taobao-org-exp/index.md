---
title: npm.taobao.org 淘宝镜像源证书过期解决方案
description: 证书过期
slug: npm-taobao-org-exp
date: 2024-08-06 08:00:00+0000
image: npm.png
categories:
  - TC
tags:
  - zhihu
  - 2024
---

## npm.taobao.org https 证书过期

- 这玩意过期也不是一两天了，所有用了依赖这个镜像源的都会直接报错。
- 前阵子因为查构建失败问题，折腾了好几个小时才意识到是证书过期了。
- 一开始以为是自己的网络问题，后来发现是源的问题。

虽然这个域名已经被cname到 `registry.npmmirror.com`，然而在npm的时候，并不会自动跳转到这个新的域名。

So，依旧构建失败。

## 解决方案

```bash
npm config set registry registry.npmmirror.com
npm config set registry registry.npmjs.org
```

如果项目根目录有 `.npmrc` 文件，记得要改下这个文件。

```bash
registry=https://registry.npmmirror.com
```

## 切记

- 不要过度依赖某厂商的服务，尤其是国内的。
- 有时候，不如直接用官方源。