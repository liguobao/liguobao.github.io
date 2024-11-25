---
title: Gitlab 低版本迁移简易教程
description: 
slug: gitlab-migration-low-version
date: 2024-11-25 08:00:00+0000
# image: girl.png
categories:
  - linux
tags:
  - linux
  - Git
  - Gitlab
---

最近在折腾GitLab迁移，由于原GitLab版本实在是太低了，

同时使用的是被魔改过的，完全不太可能从老版本跨几个大版本升级到最新版本，

所以最后决定写Python脚本调用Gitlab API导出用户和分组以及项目，

接着通过Gitlab API添加到新的Gitlab中，然后把源码仓库整个文件夹拷贝到新服务器，

使用Python执行shell命令，把所有的git 裸仓库 提交到新的Gitlab中。

## 导出用户、分组、项目

```python
```

## 创建用户、分组、项目

```python
```

## 提交Git 裸仓库



## 参考