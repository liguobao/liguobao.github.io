---
title: Using EF DbContext in Background Tasks (IHostedService)
date: 2018-01-01
tags:
- dotnet core
---

Pattern: create scopes in background workers and resolve DbContext from `IServiceProvider.CreateScope()` to respect scoped lifetimes.

