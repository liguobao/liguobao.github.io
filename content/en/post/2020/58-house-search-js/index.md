---
layout: post
title: AMap + 58.com Rental Map — JavaScript Notes
category: JavaScript
date: 2016-10-04
tags:
- asp.net core
- 58City
---

Online: http://codelover.link:8080/

Repo: https://github.com/liguobao/58HouseSearch

This post highlights AMap JS APIs used in the rental map project:

- Map initialization via `new AMap.Map(...)`
- IP location with `AMap.CitySearch()` and `map.setBounds(...)`
- Auto‑recenter: listen for `map.on('moveend', ...)` and call `map.getCity()`
- Converting city names to 58.com subdomains (e.g., 上海→sh, 北京→bj), using a scraped mapping instead of naive pinyin initials

Code snippets remain as in the Chinese version (see original for details). The key idea is to keep the map state in sync with user context (IP/city/center), and normalize city identifiers for downstream requests to 58.com.

