---
layout: post
title: A DI Misstep That Cascaded into Startup Incidents
category: APM
date: 2020-06-07
tags:
  - DI
  - .NET Core
  - Async
---

Symptoms: on startup, the first wave of traffic produced no responses while health checks looked OK, so the k8s LB kept routing, requests piled up, DB connections spiked, and pods flapped.

What it wasn’t: not a DB bottleneck (CPU/IO fine), not EF per se, not external infra issues.

Contributing factors and fixes:

- Warm up DB connections and EF metadata (limited effect).
- Make controller/service code truly async end‑to‑end (notably improved p50/p95 and reduced startup pileups).
- Pre‑warm instances before cutting traffic over to reduce cold‑start jitter.
- Be careful with filters that do network/Redis/MaxMind work during request entry; ensure these calls are async, cached, and bounded.

Root cause: a dependency injected component with expensive sync work on the hot path during startup. Fixing the DI lifetime and making the path fully async eliminated the cascade.

