---
title: VS Code 支持 C# 解决方案文件
description: VSCode,Csharp,解决方案文件
slug: vscode-support-csharp-sln
date: 2024-06-10 08:00:00+0000
image: code.png
categories:
  - TC
tags:
  - zhihu
  - 2024
---

## VS Code的C#插件好像不太正常

最近起了个新的C#项目，想着多个Project 按文件夹划分，

在根目录扔一个solution就完事了，但是在VS Code里面打开的时候，

发现在Project 源码里面引用信息和提示都不太正常，

迷，一直都是这样用的，不应该有什么问题嘛。

搜了一波发现，有些朋友也遇到了类似的问题。

- [[VS Code] 解决C#代码F12无效](https://www.cnblogs.com/WikiChen/p/17061707.html)
- [VSCode C# 转到定义(F12)和ctrl+鼠标左键无效的解决方案](https://blog.csdn.net/s5260203/article/details/120012240)

说到的基本都是，OmniSharp插件不太正常，需要手动指定一下。

但是...

[omnisharp-vscode:This repository has been archived by the owner on Jun 22, 2023. It is now read-only.](https://github.com/github/omnisharp-vscode)

自然，这个方法也是没用的了。

不过嘛。

[dotnet/vscode-csharp](https://github.com/dotnet/vscode-csharp) 已经是被官方支持的了。

A Visual Studio Code extension that provides rich language support for C# and is shipped along with C# Dev Kit. Powered by a Language Server Protocol (LSP) server, this extension integrates with open source components like Roslyn and Razor to provide rich type information and a faster, more reliable C# experience.


所以，是不是说应该安装一下这玩意就好了？

[C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)

最后验证了一下，确实是这样的。

C# Dev Kit 插件支持了sln的加载，安装好了之后手动确认哪个sln文件就可以了。

一切回归正常。

水完。
