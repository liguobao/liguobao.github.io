---
title: How to Copy Some Hard-to-Copy Content on Zhihu
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

Instead of bothering others for help (for free), let’s solve this with a tiny bit of code.

Open the target Zhihu answer page, e.g.:

https://www.zhihu.com/question/647193565/answer/3469007201

Open DevTools (F12), go to the console, and run:

```javascript
document.getElementsByClassName("RichContent RichContent--unescapable")[0].innerText
```

You should see the text content printed.

Right‑click the text output → “Copy string contents”, then paste wherever you need.

Done.

