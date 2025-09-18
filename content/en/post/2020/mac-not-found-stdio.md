---
layout: post
title: macOS Installing pycrypto — Clang ‘stdio.h’ Not Found
category: Mac OS
date: 2019-05-09
tags:
- Mac OS
- clang
---

# macOS Installing pycrypto — Clang ‘stdio.h’ Not Found

While installing a Python package that builds a C extension with Clang, I kept seeing:

```log
clang-4.0: warning: argument unused during compilation: '-L/usr/local/lib' [-Wunused-command-line-argument]
src/_fastmath.c:29:10: fatal error: 'stdio.h' file not found
#include <stdio.h>
         ^~~~~~~~~
1 error generated.
error: command 'clang' failed with exit status 1
```

Checked Clang:

```sh
clang --version
clang version 4.0.1 (tags/RELEASE_401/final)
Target: x86_64-apple-darwin18.5.0
InstalledDir: /usr/local/opt/llvm@4/bin
```

Looks fine… Tried setting env:

```sh
export "CFLAGS=-I/usr/local/include -L/usr/local/lib"
```

No luck. From Stack Overflow, the fix was to install SDK headers:

```sh
cd /Library/Developer/CommandLineTools/Packages
open macOS_SDK_headers_for_macOS_xx.pgk
# choose the pkg matching your macOS version
```

Run the installer UI, proceed… It works. Then `pip install pycrypto` succeeds.

