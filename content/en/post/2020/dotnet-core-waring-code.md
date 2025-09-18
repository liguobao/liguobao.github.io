---
layout: post
title: .NET Core “Async Tuning” — A Tale of Cold Starts and Pileups
category: dotnet core
date: 2019-08-03
tags:
- dotnet core
- performance
---

## Foreword

Service: .NET Core API + EF, from ~1k DAU to several tens of thousands. 6–8 Docker instances; QPS ≈ 100 overall. Per‑instance concurrency isn’t huge.

Issue: after starting and receiving a burst of traffic, requests pile up. Logs show requests arriving but not reaching business logic or DB — they seem to await scheduling. Sometimes instances crash on start; occasionally later when a sudden surge arrives.

Knowns:

- Crashes mostly at startup under burst; sometimes later under spikes.
- Instance CPU/RAM look fine; host is healthy; no heavy I/O; network is normal.
- Before the issue, DB connections are normal. As instances flap or hang, DB connections spike and can exhaust the pool.

### Guess 1: Database trouble?

EF first‑use warm‑up is slow; many first‑call queries might hit DB concurrently. Tried pre‑warming DbContext pool at startup:

```csharp
static void InitDataContextService(IApplicationBuilder app)
{
  Console.WriteLine($"Init start: {DateTime.Now}");
  var tasks = new List<Task>();
  for (var i = 0; i <= 10; i++)
  {
    using var scope = app.ApplicationServices.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<XXXContext>();
    tasks.Add(context.Database.CanConnectAsync());
  }
  Task.WaitAll(tasks.ToArray());
  Console.WriteLine($"Init finish: {DateTime.Now}");
}
```

Effect: negligible.

### Guess 2: Sync calls block resources → pileups?

Early code used synchronous controllers/services. Switched to `async` end‑to‑end:

```csharp
[HttpGet("v1/books/{id}", Name = "GetBook")]
public async Task<IActionResult> GetBook([FromRoute] string id)
  => Ok(new { data = await _bookService.FindBookDetailForUser(id), code = 0 });
```

Result: ad‑hoc tests showed ~30–50% gains; pileups reduced.

### “Warming helps” (anecdotally)

Pre‑warm instances before cutting traffic: fewer startup crashes; more stable startup. Temporary mitigation.

### Regression from a filter

One iteration worsened crashes. The largest change was an ActionFilter that parsed `X-Forwarded-For`, looked up location in Redis, fell back to MaxMind DB, then wrote back to Redis:

```csharp
public override void OnActionExecuting(ActionExecutingContext ctx)
{
  var dic = new Dictionary<string,string>();
  string ip = ctx.HttpContext.Request.Headers.GetHeaderValue("X-Forwarded-For");
  if (ip.Contains(',')) ip = ip.Split(',')[0];
  var iso = _redis.ReadHashValueByKey<string>(ConstRedisKey.IPLocations, ip)
          ?? _maxMind.GetIPLocationISOCode(ip).Tap(x => _redis.WriteHash(ConstRedisKey.IPLocations, ip, x));
  dic.Add(CommonConst.LocationHeaderKey, iso);
  _httpHeaderTools.CurrentXHeaders = new ThreadLocal<Dictionary<string,string>>(() => dic);
}
```

This added synchronous I/O on the hot path; under cold start and bursty load, it exacerbated pileups.

## Takeaways

- Make the entire request path truly async; avoid sync I/O and heavy work in filters/middleware.
- Pre‑warm critical pools/metadata to reduce cold‑start jitter.
- Measure under realistic burst/cold‑start scenarios.

