---
layout: post
title: .NET Core乱糊代码之"异步调差性能"指北
category: dotnet core
date: 2019-08-03
tags:
- dotnet core
- 性能调优
---

# .NET Core乱糊代码之"异步调差性能"指北

## 前言

故事要从好久之前说起,线上某服务从零到上线都是我撸的, 架构主要是.NET Core API + EF, 从最早的日活一千到现在日活几万.

平时在线人数几百, 高峰上千, 靠着6-8个Docker实例基本撑住了, 请求量来说QPS在100左右. 平摊下来其实每个实例的并发量不算太高.

但是某个迭代开始发现, 

这个Web API有一定几率在启动的时候接收到大量请求后堆积起来, 看日志显示请求进来了, 但是一直没有到逻辑代码或者数据库查询,

所有的请求看起来都是在等调度.

## 已知信息

- 出现场景大多是刚启动的时候大量请求进来, 可能会把实例打崩

- 偶尔有场景是突然涌进大量用户, 也可能把实例打崩

- 发生问题时实例CPU占用率和内存都很正常, 整个宿主机运转正常, 也没有大量IO读写操作, 基本排除外部因素

- 网络情况正常, 内网中没有奇怪的数据请求,不存在网络风暴之类的东西

- 发生问题前此时数据库连接数正常, 实例不断被重启后或者一直僵死中, 数据库连接数会不断增加, 也有一定几率会把数据库连接数撑爆.

## 开始"侦查模式"

### 猜想一, 数据库搞事了?

最早怀疑是数据库连接的问题.

总所周知EF首次启动特别慢, 如果一开始查询比较多进来, 直接落到数据库查询.

每个EF实例初始化都需要耗费一定时间, 这样势必是会影响整个性能的.

这种情况下, 如果可以对EF DB Pool做一次预热是不是会好一些呢?

所以曾经在Startup.cs下面写过类似的预热代码.

```csharp

private static void InitDataContextService(IApplicationBuilder app)
{
    Console.WriteLine($"InitDataContextService start,now:{DateTime.Now.ToString()}");
    var tasks = new List<Task>();
    for (var i = 0; i <= 10; i++)
    {
        var serviceScope = app.ApplicationServices.CreateScope();
        var context = serviceScope.ServiceProvider.GetRequiredService<XXXContext>();
        var t = context.Database.CanConnectAsync();
        tasks.Add(t);
    }
    Task.WaitAll(tasks.ToArray());
    Console.WriteLine($"InitDataContextService finish,now:{DateTime.Now.ToString()}");
}

```

用Task异步的方式来控制每次尽量产生不同的实例, 已达到预热数据量连接池的效果.

是不是很机智啊? (P, 然而并没有半毛钱的效果)

### 猜想二, 同步请求等待资源导致性能变差, 启动的时候被打崩?

最早版本所有的Controller Action 都是同步请求, 来一个请求同步查询数据, 执行HTTP请求等等都是正常的逻辑代码.

从来不用async/awati之类的东西.(就是这么懒)

既然出事了, 改一版看看.

开始锵锵锵改Controller Action, 改成Task<ActionResult>的形式.

基本代码

```csharp

[HttpGet("v1/books/{id}", Name = "GetBook")]
[SwaggerResponse(200, "获取成功", typeof(BookDetail))]
[SwaggerResponse(404, "找不到书籍")]
public async Task<IActionResult> GetBook([FromRoute]string id)
{
    return Ok(new { data = await _bookService.FindBookDetailForUser(id), code = 0 });
}
```

service层的用到的方法同时也改造成异步, 从头到尾都是异步.

这样测下来, 性能比同步大概涨了30%-50%左右(瞎测的, 也不太记得了数值了,没有参考意义)

就这样又上了一个版本, 比再前面几个版本都好点了, 现在发生打崩的情况降低了很多.

### 玄学现场:又做了一些奇怪的事情

后来又发现, 每次启动的时候, 在正式流量打入之前, 尽量预热一下api实例的话, 被直接打崩的几率会变低, 

整体api会比较正常启动.

至少算是一个能操作的事情, 这个时候就很傻了, 每次发布的时候都手动ab test一下还没有转入流量的实例, 

尽可能预热一下数据库和这个HTTP管道.

这个时候真的是玄学现场, 太让我沮丧了.

### 墨菲定律总是会来的

前天晚上发布新版本, 预发布环境政策, 性能正常之后推到生产, 然后全线告警.

神奇了, 现在又见鬼了.

用上面的压测预热折腾了一个小时, 没有成功全部启动应用, 最后无奈先回滚了, 回滚后服务正常.

OK, 这个迭代的代码加剧了应用被打崩的情况.

现总结一下当前情况

- 这次上线前, 数据库已经升级了配置, 整体监控上数据库没有任何的瓶颈

- 没有大的逻辑变动, 老的核心接口基本都异步改造完成, 新接口基本都是异步的

- 不存在缓存穿透问题, Redis缓存命中正常.

这次发布最大变更是IP定位, 需要处理headers中的IP数据, 

使用ActionFilterAttribute在请求进入Action方法前完成IP到地区的转换.

这里主要会用到Redis和MaxMind.Db, 优先从Redis查询IP地区缓存, 没有命中则直接查询MaxMind.Db数据, 查询好之后再写入到Redis中.

代码大概是这样的.

```csharp

public class HTTPHeaderAttribute : ActionFilterAttribute
    {

        private readonly HTTPHeaderTools _httpHeaderTools;

        private readonly MaxMindDBClient _maxMindDBClient;

        private readonly RedisService _redisService;

        public HTTPHeaderAttribute(HTTPHeaderTools httpHeaderTools,
         MaxMindDBClient maxMindDBClient, RedisService redisService)
        {
            _httpHeaderTools = httpHeaderTools;
            _maxMindDBClient = maxMindDBClient;
            _redisService = redisService;
        }



        public override void OnActionExecuting(ActionExecutingContext context)
        {
            var dicXHeaders = new Dictionary<string, string>();
            string locationISOCode = GetLocationISOCode(context);
            dicXHeaders.Add(CommonConst.LocationHeaderKey, locationISOCode);
            _httpHeaderTools.CurrentXHeaders = new ThreadLocal<Dictionary<string, string>>(() => dicXHeaders);
        }

        private string GetLocationISOCode(ActionExecutingContext context)
        {
            string requestIP = context.HttpContext.Request.Headers.GetHeaderValue("X-Forwarded-For");
            if(requestIP.Contains(","))
            {
                requestIP = requestIP.Split(",")[0];
            }
            var locationISOCode = _redisService.ReadHashValueByKey<string>(ConstRedisKey.IPLocations, requestIP);
            if (string.IsNullOrEmpty(locationISOCode))
            {
                locationISOCode = _maxMindDBClient.GetIPLocationISOCode(requestIP);
                _redisService.WriteHash(ConstRedisKey.IPLocations, requestIP, locationISOCode);
            }
            return locationISOCode;
        }
    }

```

代码先扔着, 看起来没什么特别的地方.

好了, 我要睡了明天继续...

等我睡醒了再继续..


<br/>
<br/>
<br/>
<br/>


<br/>
<br/>
<br/>
<br/>



<br/>
<br/>
<br/>
<br/>



<br/>
<br/>
<br/>
<br/>



<br/>
<br/>
<br/>
<br/>



<br/>
<br/>
<br/>
<br/>


















先直接甩真正解决问题的的代码:

```csharp
    public class HTTPHeaderAttribute : ActionFilterAttribute
    {

        private readonly HTTPHeaderTools _httpHeaderTools;

        private readonly MaxMindDBClient _maxMindDBClient;

        private readonly RedisService _redisService;

        public HTTPHeaderAttribute(HTTPHeaderTools httpHeaderTools, MaxMindDBClient maxMindDBClient, RedisService redisService)
        {
            _httpHeaderTools = httpHeaderTools;
            _maxMindDBClient = maxMindDBClient;
            _redisService = redisService;
        }



        public override async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
        {
            var dicXHeaders = new Dictionary<string, string>();
            string locationISOCode = await GetLocationISOCode(context);
            dicXHeaders.Add(CommonConst.LocationHeaderKey, locationISOCode);
            _httpHeaderTools.CurrentXHeaders = new ThreadLocal<Dictionary<string, string>>(() => dicXHeaders);
            await next();
        }

        private async Task<string> GetLocationISOCode(ActionExecutingContext context)
        {
            string requestIP = context.HttpContext.Request.Headers.GetHeaderValue("X-Forwarded-For");
            if (requestIP.Contains(","))
            {
                requestIP = requestIP.Split(",")[0];
            }
            if (string.IsNullOrEmpty(requestIP))
            {
                return CommonConst.ChinaISOCode;
            }
            var locationISOCode = await _redisService.ReadHashValueByKeyAsync(ConstRedisKey.IPLocations, requestIP);
            if (string.IsNullOrEmpty(locationISOCode))
            {
                locationISOCode = _maxMindDBClient.GetIPLocationISOCode(requestIP);
                _redisService.WriteHash(ConstRedisKey.IPLocations, requestIP, locationISOCode);
            }
            return locationISOCode;
        }
    }

```

先睡觉了, 有想法的朋友欢迎评论打架...


