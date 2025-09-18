---
layout: post
title: Bookmark Memorandum
category: Other
date: 2016-10-04
tags:
- memorandum
---

# Bookmark Memorandum

## Markdown references

- Markdown (Simplified Chinese): http://wowubuntu.com/markdown/
- Markdown Basics (quick start): http://wowubuntu.com/markdown/basic.html

## Build a free static blog (GitHub + Hexo)

- Guide: http://wsgzao.github.io/post/hexo-guide/

Install Hexo

```sh
npm install hexo-cli -g
npm install hexo --save

# If npm is slow, try the taobao registry
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

Install Hexo plugins

```sh
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-server --save
npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save
npm install hexo-renderer-marked@0.2 --save
npm install hexo-renderer-stylus@0.2 --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save
```

## Git tips

Fix “Filename too long” on Windows

```sh
git config --global core.longpaths true
```

Git + SSH usage: https://segmentfault.com/a/1190000002645623

