---
layout: post
title: Debug PHP with Visual Studio Code on Windows
category: Visual Studio
date: 2017-03-17 00:00:00
tags:
- Visual Studio
- PHP
- Debug
---
# Debugging PHP in VS Code (Windows)

macOS guide: https://zhuanlan.zhihu.com/p/37128419

Stack: VS Code + PHP Debug extension + phpStudy + Xdebug.

## Install VS Code

Download from https://code.visualstudio.com/ and install.

## Install PHP Debug extension

Search for “PHP Debug” in the Marketplace and install it.

## phpStudy

Use phpStudy for a quick PHP + Apache + MySQL stack. Switch PHP versions as needed.

Verify phpMyAdmin opens and connects locally. Then pick your target PHP version.

## Enable Xdebug

phpStudy ships with Xdebug. Open php.ini (Other options → Open config file → php-ini) and add:

```
xdebug.remote_enable = 1
xdebug.remote_autostart = 1
zend_extension="C:\\phpStudy\\php\\php-5.5.38\\ext\\php_xdebug.dll"
```

Restart phpStudy services.

## VS Code settings and launch config

Set `php.validate.executablePath` to your php.exe path (from phpStudy). Then add a PHP debug configuration (Listen for Xdebug) in launch.json.

Start debugging, set breakpoints, and hit the target page — VS Code should break into the PHP code. Use F10/F11/F5 as usual.

Notes for other stacks: configure php.ini, install the matching Xdebug DLL, and set `php.validate.executablePath` correctly.

