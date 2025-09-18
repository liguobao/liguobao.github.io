
---
title: 如何在知乎复制一些不那么好复制的内容~
description:
slug: how-to-copy-zhihu-content
date: 2024-04-27 08:00:00+0000
image: gqj.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---


但是，麻烦别人总是不太好的，毕竟我也没给钱....

反正技术问题，那么就用技术来解决呗。

没什么神奇的玩意，就一句代码。



打开对应的回答页面，如

[假定所有程序员写的代码都不出bug， 会发生什么？](https://www.zhihu.com/question/647193565/answer/3469007201)


在这个页面，F12打开浏览器控制台，输入下面这一行代码。

```javascript

document.getElementsByClassName("RichContent RichContent--unescapable")[0].innerText

```

然后大概能看到。


对着文本右键，选择 “Copy string contents”，然后粘贴到你想要粘贴的地方。

完事。