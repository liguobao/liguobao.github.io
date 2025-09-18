---
layout: post
title: Mac OS X 安装 pycrypto报 Clang not found 'stdio.h' error
category: Mac OS
date: 2019-05-09
tags:
- Mac OS
- clang
---

#   Mac OS X 安装 pycrypto报 Clang not found 'stdio.h' error

装一个Python包的时候用到了Clang编译C库, 然后一直报

```log

    clang-4.0: warning: argument unused during compilation: '-L/usr/local/lib' [-Wunused-command-line-argument]
    src/_fastmath.c:29:10: fatal error: 'stdio.h' file not found
    #include <stdio.h>
             ^~~~~~~~~
    1 error generated.
    error: command 'clang' failed with exit status 1
```

见鬼, 这不是C最通用的库么? 咋还能找不到.

检查一下clang版本.

```sh

➜  src git:(develop) clang --version                                       
clang version 4.0.1 (tags/RELEASE_401/final)Target: x86_64-apple-darwin18.5.0
Thread model: posix
InstalledDir: /usr/local/opt/llvm@4/bin

```

挺正常的...

有人说坏境变量问题, 导入一下试试.

```sh
export "CFLAGS=-I/usr/local/include -L/usr/local/lib" 
```

无果.

又翻了一遍stackoverflow..

找到了这个.

```sh
cd /Library/Developer/CommandLineTools/Packages
open macOS_SDK_headers_for_macOS_xx.pgk
# macOS_SDK_headers_for_macOS_xx.pgk 对应自己的系统版本
```

跳出安装界面, 点点点...

It work!


愉快安装 pip install pycrypto.
