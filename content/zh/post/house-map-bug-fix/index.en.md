---
title: "House-Map: Bug Fixes After a Long Hiatus"
description: Fixed issues in the House Search Map project
slug: house-map-bug-fix
date: 2024-01-13 23:09:00+0000
---

The last update was around late 2022. No strong new ideas since then; I kept occasional data maintenance going (powered by love, as always).

Last month, a certain site removed an old encryption routine, and data volume dropped a lot. A few days ago, we restored scraping for that site’s group data.

Recently, I saw user feedback in the Mini Program console saying “Copy link” stopped working.

Checked with a front-end friend — turns out the Mini Program framework we used no longer has docs. Another framework sunset… classic. Generally, avoid KPI‑driven big‑company frameworks when you can.

After some effort we got the project running. It looks like the privacy terms changed, and the clipboard API we used was disabled. So a light update should do.

Also fixed: auto-refresh when entering the list page; cache hit logic, which improved response speed. Multi-layer caching is “maxed out” here — even I got confused at times.

If time permits in a few weeks, I’ll start a framework migration and maybe a redesign. Still thinking about product direction.

I also hope to spin up an NLP text classifier: first pass to detect ad posts, then a scoring model. Might take a while — target is before mid‑year.

I tested pulling some xhs data today; the pipeline works — though whether that’s “safe” is another story :-)

That’s it for now. See you in the next update… or a farewell. Such is life. Bye.

Original (Chinese):
https://mp.weixin.qq.com/s/4nC2zZm2a6Tn4LL7H-oC8w?poc_token=HIyRtmWjVEi3VvkbLmrA_rAIxqQfWOzfTl-rWZo6

