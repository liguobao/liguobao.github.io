---
title: "npm.taobao.org Taobao Mirror Source Certificate Expiration Solution"
description: "Certificate Expiration"
slug: npm-taobao-org-exp
date: 2024-08-06 08:00:00+0000
image: npm.png
categories:
  - TC
tags:
  - zhihu
  - 2024
---

## npm.taobao.org https Certificate Expiration

- This thing has been expired for more than a day or two, all who used this mirror source will directly report error.
- Recently because of checking build failure issue, tinkered for several hours before realizing it's certificate expired.
- At first thought it was my network problem, later found it's the source problem.

Although this domain has been CNAME to `registry.npmmirror.com`, however when using npm, it won't automatically jump to this new domain.

So, still build failure.

## Solution

```bash
npm config set registry registry.npmmirror.com
npm config set registry registry.npmjs.org
```

If project root has `.npmrc` file, remember to change this file.

```bash
registry=https://registry.npmmirror.com
```

## Remember

- Don't overly rely on certain vendor's services, especially domestic ones.
- Sometimes, better to directly use official source.
