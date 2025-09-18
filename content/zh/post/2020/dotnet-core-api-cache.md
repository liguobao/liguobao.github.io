---
layout: post
title: .NET Core教程--给API加一个服务端缓存啦
category: dotnet core
date: 2019-05-05
tags:
- dotnet core
- redis
---

# .NET Core教程--给API加一个服务端缓存啦

以前给API接口写缓存基本都是这样写代码:
```C#
// redis key 
var bookRedisKey = ConstRedisKey.RecommendationBooks.CopyOne(bookId);
// 获取缓存数据         
var cacheBookIds = _redisService.ReadCache<List<string>>(bookRedisKey);
if (cacheBookIds != null)
{
    // return
}
else
{
   // 执行另外的逻辑获取数据, 然后写入缓存
}

````

然后把这一坨坨代码都散落在每个地方。



某一天，突然想起我这边的缓存基本时间都差不多，而且都是给Web API用的，

直接在API层支持缓存不就完事了。



所以， 这里用什么来做呢。



在.NET Core Web API这里的话, 两种思路:Middleware 或者ActionFilter.

不了解的同学可以看下面的文档:

[ASP.NET Core 中文文档 第四章 MVC（4.3）过滤器](https://www.cnblogs.com/dotNETCoreSG/p/aspnetcore-4_4_3-filters.html)

[ASP.NET Core 中文文档 第三章 原理（2）中间件](http://www.cnblogs.com/dotNETCoreSG/p/aspnetcore-3_2-middleware.html)



基于我这边只是部分接口支持缓存的话, 直接还是用ActionFilter实现就可以.

没撒说的, 直接上代码.

```C#
using System;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Newtonsoft.Json.Linq;

namespace XXXAPI.Filters
{
    public class DefaultCacheFilterAttribute : ActionFilterAttribute
    {
        // 这个时间用于给子类重写,实现不同时间级别的缓存
        protected TimeSpan _expireTime;
     
        // redis读写的类,没撒看的
        private readonly RedisService _redisService;

        public DefaultCacheFilterAttribute(RedisService redisService)
        {
            _redisService = redisService;

        }

        public override void OnActionExecuting(ActionExecutingContext context)
        {
            if (context.HttpContext.Request.Query.ContainsKey("refresh"))
            {
                return;
            }
            KeyConfig redisKey = GetRequestRedisKey(context.HttpContext);
            var redisCache = _redisService.ReadCache<JToken>(redisKey);
            if (redisCache != null)
            {
                context.Result = new ObjectResult(redisCache);
            }
            return;
        }

        public override void OnActionExecuted(ActionExecutedContext context)
        {
            KeyConfig redisKey = GetRequestRedisKey(context.HttpContext);
            var objResult = (ObjectResult)context.Result;
            if (objResult == null)
            {
                return;
            }
            var jToken = JToken.FromObject(objResult.Value);
            _redisService.WriteCache(redisKey, jToken);
        }

        private KeyConfig GetRequestRedisKey(HttpContext httpContext)
        {
            var requestPath = httpContext.Request.Path.Value;
            if (!string.IsNullOrEmpty(httpContext.Request.QueryString.Value))
            {
                requestPath = requestPath + httpContext.Request.QueryString.Value;
            }
            if (httpContext.Request.Query.ContainsKey("refresh"))
            {
                if (httpContext.Request.Query.Count == 1)
                {
                    requestPath = requestPath.Replace("?refresh=true", "");
                }
                else
                {
                    requestPath = requestPath.Replace("refresh=true", "");
                }
            }
            // 这里也就一个redis key的类
            var redisKey = ConstRedisKey.HTTPRequest.CopyOne(requestPath);
            if (_expireTime != default(TimeSpan))
            {
                redisKey.ExpireTime = _expireTime;
            }
            return redisKey;
        }
    }

    public static class ConstRedisKey
    {
        public readonly static KeyConfig HTTPRequest = new KeyConfig()
        {
            Key = "lemon_req_",
            ExpireTime = new TimeSpan(TimeSpan.TicksPerMinute * 30),
            DBName = 5
        };
    }

    public class KeyConfig
    {
        public string Key { get; set; }

        public TimeSpan ExpireTime { get; set; }

        public int DBName { get; set; }


        public KeyConfig CopyOne(string state)
        {
            var one = new KeyConfig();
            one.DBName = this.DBName;
            one.Key = !string.IsNullOrEmpty(this.Key) ? this.Key + state : state;
            one.ExpireTime = this.ExpireTime;
            return one;
        }

    }
}

```

然后使用的地方, 直接给Controller的Action方法加上注解即可.

如:

```C#
        [HttpGet("v1/xxx/latest")]
        [ServiceFilter(typeof(DefaultCacheFilterAttribute))]
        public IActionResult GetLatestList([FromQuery] int page = 0, [FromQuery]int pageSize = 30)
        {
            return Ok(new
            {
                data = _service.LoadLatest(page, pageSize),
                code = 0
            });
        }
```

完事...













哦, 记得在Startup.cs注入 DefaultCacheFilterAttribute.

这个注入就不用我来写的吧.







美中不足的地方在于暂时还不知道怎么直接在注解上面支持自定义缓存时间,

凑合先用了.





完结, 拜.....