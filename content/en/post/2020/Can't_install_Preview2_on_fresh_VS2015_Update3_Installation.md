---
layout: post
title : .NET Core 1.0.0 VS2015 Tooling Preview 2 — 0x80070003
category: dotnet core
date: 2016-10-04
tags:
- dotnet core
- dotnet
---

Installing “Microsoft .NET Core 1.0.0 VS 2015 Tooling Preview 2” was a bumpy ride. The bootstrapper downloads MSIs during install (online installer), but links were failing (502s) and network stability made it worse.

Workaround: download required packages manually and place them where the installer expects, or use an offline installer when available. For example, `DotNetCore.1.0.1-SDK.1.0.0.Preview2-003133-x64.exe` could be fetched via a reliable mirror/tool, while dependent MSIs were trickier due to broken links at the time.

If you hit 0x80070003 errors (path not found) during install, verify the cached package paths, disable unstable mirrors, and try a fully offline SDK installer. Alternatively, move to newer .NET Core SDK versions which have stable distribution and improved installers.

