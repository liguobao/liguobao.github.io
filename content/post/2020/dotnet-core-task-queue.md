---
layout: post
title: 任务队列和异步接口的正确打开方式(.NET Core版本)
category: dotnet core
date: 2019-04-05
tags:
- dotnet core
- redis
- 消息队列
- 异步API
---

# 任务队列和异步接口的正确打开方式

## 什么是异步接口?

<h2 id="asynchronous-operations">Asynchronous Operations</h2>

Certain types of operations might require processing of the request in an asynchronous manner (e.g. validating a bank account, processing an image, etc.) in order to avoid long delays on the client side and prevent long-standing open client connections waiting for the operations to complete. For such use cases, APIs MUST employ the following pattern:

**_For `POST` requests_:**

* Return the `202 Accepted` HTTP response code.
* In the response body, include one or more URIs as hypermedia links, which could include:
    * The final URI of the resource where it will be available in future if the ID and path are already known. Clients can then make an HTTP `GET` request to that URI in order to obtain the completed resource. Until the resource is ready, the final URI SHOULD return the HTTP status code `404 Not Found`.
    
    `{ "rel": "self", "href": "/v1/namespace/resources/{resource_id}", "method": "GET" }`
    
    * A temporary request queue URI where the status of the operation may be obtained via some temporary identifier. Clients SHOULD make an HTTP `GET` request to obtain the status of the operation which MAY include such information as completion state, ETA, and final URI once it is completed.
    
    `{ "rel": "self", "href": "/v1/queue/requests/{request_id}, "method": "GET" }"`
    

**_For `PUT`/`PATCH`/`DELETE`/`GET` requests_:**

Like `POST`, you can support PUT/`PATCH`/`DELETE`/`GET` to be asynchronous. The behaviour would be as follows:

* Return the `202 Accepted` HTTP response code.
* In the response body, include one or more URIs as hypermedia links, which could include:
	* A temporary request queue URI where the status of the operation may be obtained via some temporary identifier. Clients SHOULD make an HTTP `GET` request to obtain the status of the operation which MAY include such information as completion state, ETA, and final URI once it is completed.
       
    `{ "rel": "self", "href": "/v1/queue/requests/{request_id}, "method": "GET" }"`


**_APIs that support both synchronous and asynchronous processing for an URI_:**

APIs that support both synchronous and asynchronous operations for a particular URI and an HTTP method combination, MUST recognize the [`Prefer`](index.md#http-standard-headers) header and exhibit following behavior:

* If the request contains a `Prefer=respond-async` header, the service MUST switch the processing to asynchronous mode. 
* If the request doesn't contain a `Prefer=respond-async` header, the service MUST process the request synchronously.

It is desirable that all APIs that implement asynchronous processing, also support [webhooks](https://en.wikipedia.org/wiki/Webhook) as a mechanism of pushing the processing status to the client.

资料引自:[paypal/API Design Patterns And Use Cases:asynchronous-operations](https://github.com/paypal/api-standards/blob/master/patterns.md#asynchronous-operations)

### 用人话来说

* 简单来说就是请求过来,直接返回对应的resourceId/request_id,然后可以通过resourceId/request_id查询处理结果

* 处理过程可能是队列,也可能直接是异步操作

* 如果还没完成处理,返回404,如果处理完成,正常返回对应数据

好像也没什么讲了....

全文结束吧.





































## 样例代码部分啦

### 实现逻辑

- 创建任务,生成"request-id"存储到对应redis zset队列中

- 同时往redis channel发出任务消息, 后台任务处理服务自行处理此消息(生产者-消费者模式)

- 任务处理服务处理完消息之后,将处理结果写入redis,request-id为key,结果为value,然后从从redis zset从移除对应的"request-id"

- 获取request-id处理结果时:如果request-id能查询到对应的任务处理结果,直接返回处理完的数据; 如果request-id还在sortset队列则直接返回404 + 对应的位置n,表示还在处理中,前面还有n个请求;

时序图大概长这样:

![](https://ws1.sinaimg.cn/large/64d1e863gy1fz3r5m9x0ij20v80q277b.jpg)


### 喜闻乐见代码时间

RequestService.cs


```C#
// RequestService.cs

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using CorrelationId;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json.Linq;
using StackExchange.Redis;
using static StackExchange.Redis.RedisChannel;

namespace MTQueue.Service
{
    public class RequestService
    {


        private readonly ICorrelationContextAccessor _correlationContext;

        private readonly ConnectionMultiplexer _redisMultiplexer;

        private readonly IServiceProvider _services;

        private readonly ILogger<RequestService> _logger;

        public RequestService(ICorrelationContextAccessor correlationContext,
        ConnectionMultiplexer redisMultiplexer, IServiceProvider services,
        ILogger<RequestService> logger)
        {
            _correlationContext = correlationContext;
            _redisMultiplexer = redisMultiplexer;
            _services = services;
            _logger = logger;
        }

        public long? AddRequest(JToken data)
        {
            var requestId = _correlationContext.CorrelationContext.CorrelationId;
            var redisDB = _redisMultiplexer.GetDatabase(CommonConst.DEFAULT_DB);
            var index = redisDB.SortedSetRank(CommonConst.REQUESTS_SORT_SETKEY, requestId);
            if (index == null)
            {
                data["requestId"] = requestId;
                redisDB.SortedSetAdd(CommonConst.REQUESTS_SORT_SETKEY, requestId, GetTotalSeconds());
                PushRedisMessage(data.ToString());
            }
            return redisDB.SortedSetRank(CommonConst.REQUESTS_SORT_SETKEY, requestId);
        }

        public static long GetTotalSeconds()
        {
            return (long)(DateTime.Now.ToLocalTime() - new DateTime(1970, 1, 1).ToLocalTime()).TotalSeconds;
        }

        private void PushRedisMessage(string message)
        {
            Task.Run(() =>
            {
                try
                {
                    using (var scope = _services.CreateScope())
                    {
                        var multiplexer = scope.ServiceProvider.GetRequiredService<ConnectionMultiplexer>();
                        multiplexer.GetSubscriber().PublishAsync(CommonConst.REQUEST_CHANNEL, message);
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogError(-1, ex, message);
                }
            });
        }

        public Tuple<JToken, long?> GetRequest(string requestId)
        {
            var redisDB = _redisMultiplexer.GetDatabase(CommonConst.DEFAULT_DB);
            var keyIndex = redisDB.SortedSetRank(CommonConst.REQUESTS_SORT_SETKEY, requestId);
            var response = redisDB.StringGet(requestId);
            if (response.IsNull)
            {
                return Tuple.Create<JToken, long?>(default(JToken), keyIndex);
            }
            return Tuple.Create<JToken, long?>(JToken.Parse(response), keyIndex);
        }

    }
}
```

```C#

// RedisMQListener.cs

using System;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MTQueue.Model;
using MTQueue.Service;
using Newtonsoft.Json.Linq;
using StackExchange.Redis;
using static StackExchange.Redis.RedisChannel;

namespace MTQueue.Listener
{
    public class RedisMQListener : IHostedService
    {
        private readonly ConnectionMultiplexer _redisMultiplexer;

        private readonly IServiceProvider _services;

        private readonly ILogger<RedisMQListener> _logger;

        public RedisMQListener(IServiceProvider services, ConnectionMultiplexer redisMultiplexer,
        ILogger<RedisMQListener> logger)
        {
            _services = services;
            _redisMultiplexer = redisMultiplexer;
            _logger = logger;
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            Register();
            return Task.CompletedTask;
        }


        public virtual bool Process(RedisChannel ch, RedisValue message)
        {
            _logger.LogInformation("Process start,message: " + message);
            var redisDB = _services.GetRequiredService<ConnectionMultiplexer>()
            .GetDatabase(CommonConst.DEFAULT_DB);
            var messageJson = JToken.Parse(message);
            var requestId = messageJson["requestId"]?.ToString();
            if (string.IsNullOrEmpty(requestId))
            {
                _logger.LogWarning("requestId not in message.");
                return false;
            }
            var mtAgent = _services.GetRequiredService<ZhihuClient>();
            var text = mtAgent.GetZhuanlan(messageJson);
            redisDB.StringSet(requestId, text.ToString(), CommonConst.RESPONSE_TS);
            _logger.LogInformation("Process finish,requestId:" + requestId);
            redisDB.SortedSetRemove(CommonConst.REQUESTS_SORT_SETKEY, requestId);
            return true;
        }


        public void Register()
        {
            var sub = _redisMultiplexer.GetSubscriber();
            var channel = CommonConst.REQUEST_CHANNEL;
            sub.SubscribeAsync(channel, (ch, value) =>
            {
                Process(ch, value);
            });
        }

        public void DeRegister()
        {
            // this.connection.Close();
        }


        public Task StopAsync(CancellationToken cancellationToken)
        {
            // this.connection.Close();
            return Task.CompletedTask;
        }
    }

}

```

```C#

// RequestsController.cs

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using CorrelationId;
using Microsoft.AspNetCore.Mvc;
using MTQueue.Service;
using Newtonsoft.Json.Linq;

namespace MTQueue.Controllers
{
    [Route("v1/[controller]")]
    [ApiController]
    public class RequestsController : ControllerBase
    {

        private readonly ICorrelationContextAccessor _correlationContext;

        private readonly RequestService _requestService;

        private readonly ZhihuClient _mtAgentClient;

        public RequestsController(ICorrelationContextAccessor correlationContext,
         RequestService requestService, ZhihuClient mtAgentClient)
        {
            _correlationContext = correlationContext;
            _requestService = requestService;
            _mtAgentClient = mtAgentClient;
        }



        [HttpGet("{requestId}")]
        public IActionResult Get(string requestId)
        {
            var result = _requestService.GetRequest(requestId);
            var resource = $"/v1/requests/{requestId}";
            if (result.Item1 == default(JToken))
            {
                return NotFound(new { rel = "self", href = resource, method = "GET", index = result.Item2 });
            }
            return Ok(result.Item1);
        }

        [HttpPost]
        public IActionResult Post([FromBody] JToken data, [FromHeader(Name = "Prefer")]string prefer)
        {
            if (!string.IsNullOrEmpty(prefer) && prefer == "respond-async")
            {
                var index = _requestService.AddRequest(data);
                var requestId = _correlationContext.CorrelationContext.CorrelationId;
                var resource = $"/v1/requests/{requestId}";
                return Accepted(resource, new { rel = "self", href = resource, method = "GET", index = index });
            }
            return Ok(_mtAgentClient.GetZhuanlan(data));
        }
    }
}

```

完整代码见:[https://github.com/liguobao/TaskQueueSample](https://github.com/liguobao/TaskQueueSample)
