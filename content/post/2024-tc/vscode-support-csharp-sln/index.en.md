---
title: "VS Code Support for C# Solution Files"
description: "VSCode, C#, Solution Files"
slug: vscode-support-csharp-sln
date: 2024-06-10 08:00:00+0000
image: code.png
categories:
  - TC
tags:
  - zhihu
  - 2024
---

## The C# plugin in VS Code seems abnormal

Recently started a new C# project, thinking of dividing multiple projects by folders, putting a solution in the root directory. But when opening in VS Code, the reference information and hints in the project source code are not normal. Confused, I've always used it this way, shouldn't there be any problem.

Searched and found that some friends also encountered similar problems.

- [[VS Code] Fix C# code F12 invalid](https://www.cnblogs.com/WikiChen/p/17061707.html)
- [VSCode C# Go to Definition (F12) and Ctrl+Left Click Invalid Solution](https://blog.csdn.net/s5260203/article/details/120012240)

Basically, the OmniSharp plugin is not working properly, need to specify manually. But...

[omnisharp-vscode: This repository has been archived by the owner on Jun 22, 2023. It is now read-only.](https://github.com/github/omnisharp-vscode)

Naturally, this method is useless.

However.

[dotnet/vscode-csharp](https://github.com/dotnet/vscode-csharp) is now officially supported.

A Visual Studio Code extension that provides rich language support for C# and is shipped along with C# Dev Kit. Powered by a Language Server Protocol (LSP) server, this extension integrates with open source components like Roslyn and Razor to provide rich type information and a faster, more reliable C# experience.

So, does that mean I should install this thing?

[C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)

Finally verified, it's indeed like this. The C# Dev Kit plugin supports loading sln, after installing, manually confirm which sln file. Everything back to normal.

Done.
