---
layout: post
title: ASP.NET Core — Add Server-Side Cache via Action Filter
category: dotnet core
date: 2019-05-05
tags:
- dotnet core
- redis
---

# Server-Side Caching for APIs with an Action Filter

Instead of sprinkling cache lookups/writes across controllers, add a reusable ActionFilter that caches responses in Redis based on the request path + querystring.

```csharp
public class DefaultCacheFilterAttribute : ActionFilterAttribute
{
    protected TimeSpan _expireTime; // override in subclass for custom TTLs
    private readonly RedisService _redisService;
    public DefaultCacheFilterAttribute(RedisService redisService) => _redisService = redisService;

    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (context.HttpContext.Request.Query.ContainsKey("refresh")) return;
        var redisKey = GetRequestRedisKey(context.HttpContext);
        var cached = _redisService.ReadCache<JToken>(redisKey);
        if (cached != null) context.Result = new ObjectResult(cached);
    }

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        var obj = (ObjectResult)context.Result;
        if (obj == null) return;
        var key = GetRequestRedisKey(context.HttpContext);
        _redisService.WriteCache(key, JToken.FromObject(obj.Value));
    }

    private KeyConfig GetRequestRedisKey(HttpContext http)
    {
        var path = http.Request.Path.Value;
        if (!string.IsNullOrEmpty(http.Request.QueryString.Value)) path += http.Request.QueryString.Value;
        if (http.Request.Query.ContainsKey("refresh"))
            path = path.Replace("?refresh=true", "").Replace("refresh=true", "");
        var key = ConstRedisKey.HTTPRequest.CopyOne(path);
        if (_expireTime != default) key.ExpireTime = _expireTime;
        return key;
    }
}

public static class ConstRedisKey
{
    public static readonly KeyConfig HTTPRequest = new KeyConfig
    {
        Key = "lemon_req_",
        ExpireTime = TimeSpan.FromMinutes(30),
        DBName = 5
    };
}
```

Usage:

```csharp
[HttpGet("v1/xxx/latest")]
[ServiceFilter(typeof(DefaultCacheFilterAttribute))]
public IActionResult GetLatestList([FromQuery] int page = 0, [FromQuery] int pageSize = 30)
  => Ok(new { data = _service.LoadLatest(page, pageSize), code = 0 });
```

Register `DefaultCacheFilterAttribute` in DI (Startup). Optionally subclass to set different TTLs per action.

