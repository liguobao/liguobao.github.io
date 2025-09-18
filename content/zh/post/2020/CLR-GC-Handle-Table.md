---
layout: post
title: CLR 手动监控和控制对象的生存周期
category: dotnet
date: 2016-10-04
tags:
- dotnet core
- dotnet
---
CLR为每个 ApDomain 都提供了一个 **GC句柄表（GC Handle table）**，允许应用程序监视或者手动控制对象的生存期。这个表在 ApDomain 创建之初是空白的。

表中每个记录项都包含一下两种信息：

对托管堆中的一个对象的引用，以及之初如何监视或者控制对象的标志（flag）。


```csharp

```