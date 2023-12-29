---
title: 如何配置Github Page页面
description: 
slug: how-to-set-github-page
date: 2023-12-19 13:34:08+0000
image: cover.jpg
categories:
    - Gihtub
tags:
    - Github
    - Action
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


具体教程：[配置 GitHub Pages 站点的发布源](https://docs.github.com/zh/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)


重点是：

设置对应仓库的分支。

一般我们使用Hugo之类的模板，

都是生成了一个public 静态文件夹的。

然后使用Github Action生成静态文件，

一般是发布到当前任务的独立分支。


## 我的例子

我的仓库是：

- [iguobao/liguobao.github.io](https://github.com/liguobao/liguobao.github.io)


主分支 [master](https://github.com/liguobao/liguobao.github.io) 就是我的配置和文章，

静态文件分支是：

[liguobao.github.io/tree/gh-pages](https://github.com/liguobao/liguobao.github.io/tree/gh-pages)

这个时候，就需要在

Github 项目中 settings/pages 设置我的 "Your GitHub Pages site"

分支为 gh-pages

![](cover.png)


然后。

完事。


> Photo by [Pawel Czerwinski](https://unsplash.com/@pawel_czerwinski) on [Unsplash](https://unsplash.com/)