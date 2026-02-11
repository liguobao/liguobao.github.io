---
title: What’s the Shortest Server Rental? Can I Rent for Just a Week?
slug: how-long-can-i-buy-a-server
date: 2024-01-29 08:00:00+0000
categories:
  - Zhihu
tags:
  - 2024
---

Source: https://www.zhihu.com/question/639020341

Q: What’s the shortest I can rent a server? Can I rent for only a week?

A:

1) Is there such a service?

Yes. Choose pay‑as‑you‑go/on‑demand. Most cloud products support it. In China, many providers require real‑name verification and a prepaid balance (e.g., >100 CNY) to enable it — a bit sneaky.

2) Any “cheaper” options?

Sure: for static sites, GitHub Pages is simplest. I wrote this earlier:

How to configure a GitHub Pages site: https://liguobao.github.io/p/how-to-set-github-page/

3) I want to expose my local site to the internet — any simple solution?

Yes. Easiest is FRP (intranet penetration), but you still need a public server for the FRP server side.

How to configure FRP: https://liguobao.github.io/p/how-to-set-frp/

Alternatively, use a tunneling service. When I played with stable-diffusion-webui, it provided a temporary public URL (via https://www.gradio.app/). You can use similar services to expose a local service as a temporary public address.

Docs: Sharing Your App — https://www.gradio.app/3.50.2/guides/sharing-your-app

Have fun!

