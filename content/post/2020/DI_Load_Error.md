---
layout: post
title: 一次依赖注入不慎引发的一连串事故
category: APM
date: 2020-06-07
tags:
  - DI
  - 依赖注入
  - .NET Core
  - 异步编程
---

一次依赖注入不慎引发的一连串事故。

## 起因和现象

偶尔会看到线上服务启动的时候第一波流量进来之后，

迟迟没有任何的响应，同时服务的监控检查接口正常，

所以 K8S 集群认为服务正常，继续放入流量。

查看日志基本如下：

```log

[2020-06-05T13:00:30.7080743+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test
[2020-06-05T13:00:30.7081525+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:31.7074253+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR21" started.
[2020-06-05T13:00:31.7077051+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:31.7077942+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:32.2103440+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR22" started.
[2020-06-05T13:00:32.2118432+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:32.2125894+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-ba'lan'ce does not match a supported file type
[2020-06-05T13:00:33.2223942+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR23" started.
[2020-06-05T13:00:33.2238736+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:33.2243808+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:34.2177528+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR24" started.
[2020-06-05T13:00:34.2189073+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:34.2193483+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:35.2169806+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR25" started.
[2020-06-05T13:00:35.2178259+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:35.2181055+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:36.2183025+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR26" started.
[2020-06-05T13:00:36.2195050+00:00 Microsoft.AspNetCore.Hosting.Diagnostics INF] Request starting HTTP/1.0 GET http://172.16.2.52/v1/user/test/account-balance
[2020-06-05T13:00:36.2199702+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG] The request path /v1/user/test/account-balance does not match a supported file type
[2020-06-05T13:00:37.2373822+00:00 Microsoft.AspNetCore.Server.Kestrel DBG] Connection id "0HM09A1MAAR27" started.

```

---

### 引发的几种后果

#### 客户端调用超时

经过了 30S 甚至更长时间后看到大量的数据库连接被初始化，然后开始集中式返回。

此时可能对于客户端调用来说这一批请求都是超时的，

严重影响用户体验和某些依赖于此的其他接口。

#### 数据库连接暴涨

因为同时进入大量数据库查询请求触发数据库 DbContextPool 大量创建，

连接数随之暴涨，数据库查询性能急速下降，可能引发其他的应用问题。

#### 引发服务“雪崩”效应，服务不可用

请求堆积的情况下，

health-check 接口响应异常，

导致 k8s 主动重启服务，重启后继续上述情况，

不断恶化最后导致服务不可用。

---

## 排查问题

### 数据库的问题 ？

当然，首先怀疑的就是数据库了。

存在性能瓶颈？慢查询导致不响应？发布期间存在其他的异常？

这类的问题都意义排查起来了。

最后发现，

这种情况发生的时候，数据库监控里面一片祥和。

数据库 IO、CPU、内存都正常，

连接数暴涨是这种情况发生的时候带来的，

而不是连接数暴涨之后导致了此情况。

### 数据库驱动或者 EF Core 框架的问题？

是的，

这个怀疑一直都存在于脑海中。

最终，

昨天带着“被挨骂的情况”去问了下“Pomelo.EntityFrameworkCore.MySql”的作者。

```log

春天的熊 18:34:08
柚子啊，我这边的.NET Core服务刚起来，建立MySQL连接的时候好慢，然后同一批请求可能无法正常响应，这个有什么好的解决思路吗？

Yuko丶柚子 18:34:29
Min Pool Size = 200

Yuko丶柚子 18:34:32
放连接字符串里

春天的熊 18:34:53
这个字段支持了吗？

Yuko丶柚子 18:35:07
一直都支持

春天的熊 18:35:56
等等，      public static IServiceCollection AddDbContextPool<TContext>([NotNullAttribute] this IServiceCollection serviceCollection, [NotNullAttribute] Action<DbContextOptionsBuilder> optionsAction, int poolSize = 128) where TContext : DbContext;

春天的熊 18:36:13
这里不是默认最大的128么？

Yuko丶柚子 18:36:18
你这个pool size是dbcontext的

Yuko丶柚子 18:36:21
我说的是mysql连接字符串的

Yuko丶柚子 18:36:28
dbcontext的pool有什么用

春天的熊 18:43:13
我问个讨打的问题，dbcontext 是具体的链接实例，EF用的，Min Pool Size 指的是这一个实例上面的连接池吗“？

Yuko丶柚子 18:44:07
你在说什么。。。

Yuko丶柚子 18:45:58
放到mysql的连接字符串上

Yuko丶柚子 18:46:14
这样第一次调用MySqlConnection的时候就会建立200个连接

春天的熊 18:46:56
默认是多少来的？100吗？

Yuko丶柚子 18:48:33
0

Yuko丶柚子 18:48:40
max默认是100

Yuko丶柚子 18:52:50
DbContextPool要解决的问题你都没搞清楚

春天的熊 18:53:23
DbContextPool要解决的是尽量不去重复创建DbContext

Yuko丶柚子 18:53:34
为什么不要重复创建DbContext

春天的熊 18:53:50
因为每个DbContext创建的代价很高，而且很慢

Yuko丶柚子 18:54:01
创建DbContext有什么代价

Yuko丶柚子 18:54:03
哪里慢了

Yuko丶柚子 18:54:06
都是毫秒级的

Yuko丶柚子 18:54:20
他的代价不在于创建 而在于回收

Yuko丶柚子 18:54:25
DbContextPool要解决的问题是 因为DbContext属于较大的对象，而且是频繁被new，而且经常失去引用导致GC频繁工作。

```

Yuko 大大说的情况感觉会是一个思路，

所以第一反应就是加了参数控制连接池。

不过，无果。

5 个实例，

有 3 个实例正常启动，

2 个实例会重复“雪崩”效应，最终无法正常启动。

这个尝试操作重复了好多次，

根据文档和 Yuko 大大指导继续加了不少 MySQL 链接参数，

最后，

重新学习了一波链接参数的优化意义，

无果。

---

### 究竟数据库驱动有没有问题？

没什么好的思路了，

远程到容器里面 Debug 基本不太现实（重新打包 + 容器化打包 + k8s + 人肉和服务器垮大洋），

要不，试试把日志登录调节到 Debug 看看所有的行为？

```json
{
  "Using": ["Serilog.Sinks.Console"],
  "MinimumLevel": {
    "Default": "Debug",
    "Override": {
      "Microsoft": "Debug"
    }
  },
  "WriteTo": [
    {
      "Name": "Console",
      "Args": {
        "outputTemplate": "[{Timestamp:o}  {SourceContext} {Level:u3}] {Message:lj}{NewLine}{Exception}"
      }
    }
  ]
}
```

当然，这个事情没有直接在正常的生产环境执行。

这里是使用新配置，重新起新实例来操作。

然后我们看到程序启动的时候执行 EFMigration 的时候，

程序和整个数据库交互的完整日志。

```log

[2020-06-05T12:59:56.4147202+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Opening connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4159970+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Opened connection to database 'user_pool' on server 'a'li'yun'.
[2020-06-05T12:59:56.4161172+00:00 Microsoft.EntityFrameworkCore.Database.Command DBG] Executing DbCommand [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='user_pool' AND TABLE_NAME='__EFMigrationsHistory';
[2020-06-05T12:59:56.4170776+00:00 Microsoft.EntityFrameworkCore.Database.Command INF] Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='user_pool' AND TABLE_NAME='__EFMigrationsHistory';
[2020-06-05T12:59:56.4171630+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Closing connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4172458+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Closed connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4385345+00:00 Microsoft.EntityFrameworkCore.Database.Command DBG] Creating DbCommand for 'ExecuteReader'.
[2020-06-05T12:59:56.4386201+00:00 Microsoft.EntityFrameworkCore.Database.Command DBG] Created DbCommand for 'ExecuteReader' (0ms).
[2020-06-05T12:59:56.4386763+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Opening connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4400143+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Opened connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4404529+00:00 Microsoft.EntityFrameworkCore.Database.Command DBG] Executing DbCommand [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT `MigrationId`, `ProductVersion`
FROM `__EFMigrationsHistory`
ORDER BY `MigrationId`;
[2020-06-05T12:59:56.4422387+00:00 Microsoft.EntityFrameworkCore.Database.Command INF] Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT `MigrationId`, `ProductVersion`
FROM `__EFMigrationsHistory`
ORDER BY `MigrationId`;
[2020-06-05T12:59:56.4446400+00:00 Microsoft.EntityFrameworkCore.Database.Command DBG] A data reader was disposed.
[2020-06-05T12:59:56.4447422+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Closing connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.4447975+00:00 Microsoft.EntityFrameworkCore.Database.Connection DBG] Closed connection to database 'user_pool' on server 'aliyun-rds'.
[2020-06-05T12:59:56.5170419+00:00 Microsoft.EntityFrameworkCore.Migrations INF] No migrations were applied. The database is already up to date.
```

---

看到这里的时候，由于发现我们之前对 DbContext 和 DbConnection 的理解不太好，

想搞清楚究竟是不 db connection 创建的时候有哪些行为，

于是我们找到了 [dotnet/efcore Github ](https://github.com/dotnet/efcore)的源码开始拜读，

PS： 源码真香，能看源码真好。

尝试通过“Opening connection”找到日志的场景。

想了解这个日志输出的时候代码在做什么样的事情，可能同时会有哪些行为。

在考虑是不是其他的一些行为导致了上面的服务问题？

最终在[RelationalConnection.cs](https://github.com/dotnet/efcore/blob/14f13ef6dfb0aae7e2faedfb0de604070fc5d101/src/EFCore.Relational/Storage/RelationalConnection.cs)确认上面这些数据库相关日志肯定是会输出的，不存在其他的异常行为。

PS：不用细看，我们认真浏览了代码之后确认 DbContext 正常初始化，

```c#
        /// <summary>
        ///     Asynchronously opens the connection to the database.
        /// </summary>
        /// <param name="errorsExpected"> Indicate if the connection errors are expected and should be logged as debug message. </param>
        /// <param name="cancellationToken">
        ///     A <see cref="CancellationToken" /> to observe while waiting for the task to complete.
        /// </param>
        /// <returns>
        ///     A task that represents the asynchronous operation, with a value of <see langword="true"/> if the connection
        ///     was actually opened.
        /// </returns>
        public virtual async Task<bool> OpenAsync(CancellationToken cancellationToken, bool errorsExpected = false)
        {
            if (DbConnection.State == ConnectionState.Broken)
            {
                await DbConnection.CloseAsync().ConfigureAwait(false);
            }

            var wasOpened = false;
            if (DbConnection.State != ConnectionState.Open)
            {
                if (CurrentTransaction != null)
                {
                    await CurrentTransaction.DisposeAsync().ConfigureAwait(false);
                }

                ClearTransactions(clearAmbient: false);
                await OpenDbConnectionAsync(errorsExpected, cancellationToken).ConfigureAwait(false);
                wasOpened = true;
            }

            _openedCount++;

            HandleAmbientTransactions();

            return wasOpened;
        }


        private async Task OpenDbConnectionAsync(bool errorsExpected, CancellationToken cancellationToken)
        {
            var startTime = DateTimeOffset.UtcNow;
            var stopwatch = Stopwatch.StartNew();

            // 日志输出在这里
            var interceptionResult
                = await Dependencies.ConnectionLogger.ConnectionOpeningAsync(this, startTime, cancellationToken)
                    .ConfigureAwait(false);

            try
            {
                if (!interceptionResult.IsSuppressed)
                {
                    await DbConnection.OpenAsync(cancellationToken).ConfigureAwait(false);
                }
                // 日志输出在这里
                await Dependencies.ConnectionLogger.ConnectionOpenedAsync(this, startTime, stopwatch.Elapsed, cancellationToken)
                    .ConfigureAwait(false);
            }
            catch (Exception e)
            {
                await Dependencies.ConnectionLogger.ConnectionErrorAsync(
                    this,
                    e,
                    startTime,
                    stopwatch.Elapsed,
                    errorsExpected,
                    cancellationToken)
                    .ConfigureAwait(false);

                throw;
            }

            if (_openedCount == 0)
            {
                _openedInternally = true;
            }
        }
```

当然，我们同时也去看了一眼 [MySqlConnector](https://mysqlconnector.net/)的源码，

确认它自身是维护了数据库连接池的。到这里基本确认不会是数据库驱动导致的上述问题。

---

## 某种猜测

### 肯定是有什么奇怪的行为阻塞了当前服务进程，

### 导致数据库连接的日志也没有输出。

### 锁？ 异步等同步？资源初始化问题？

周五晚上查到了这里已经十一点了，

于是先下班回家休息了。

于是，

周六练完车之后 Call 了一下小伙伴，

又双双开始了愉快的 Debug。

---

### 插曲

小伙伴海林回公司前发了个朋友圈。

“ 咋们继续昨天的 bug，

特此立 flag：修不好直播吃 bug

反正不是你死就是我亡…”

我调侃评论说:

你等下，我打包下代码去楼下打印出来待会当晚饭

---

## 开始锁定问题

### 中间件导致的吗？

```log
[2020-06-05T13:00:35.2181055+00:00 Microsoft.AspNetCore.StaticFiles.StaticFileMiddleware DBG]

The request path /v1/user/test/account-balance does not match a supported file type

```

我们对着这个日志思考了一会人生，

然后把引用此中间件的代码注释掉了，

不过，无果。

### 自定义 filters 导致的吗？

```log

[2020-06-05T13:01:05.3126001+00:00 Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker DBG] Execution plan of exception filters (in the following order): ["None"]
[2020-06-05T13:01:05.3126391+00:00 Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker DBG] Execution plan of result filters (in the following order): ["Microsoft.AspNetCore.Mvc.ViewFeatures.Filters.SaveTempDataFilter", "XXX.Filters.HTTPHeaderAttribute (Order: 0)"]
[2020-06-05T13:01:05.3072206+00:00 Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker DBG] Execution plan of authorization filters (in the following order): ["None"]
```

看到这个日志我们考虑了一下，

是不是因为 filters 导致了问题。

毕竟在 HTTPHeaderAttribute 我们还还做了 ThreadLocal<Dictionary<string, string>> CurrentXHeaders

这里怀疑是不是我们的实现存在锁机制导致“假死问题”。

尝试去掉。

不过，

无果。

---

### 尝试使用 ptrace

没什么很好的头绪了，要不上一下 ptrace 之类的工具跟一下系统调用？

最早在去年就尝试过使用 ptrace 抓进程数据看系统调用，

后来升级到.NET Core3.0 之后，官方基于 Events + LTTng 之类的东西做了 dotnet-trace 工具，

官网说明：[dotnet-trace performance analysis utility](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)

改一下打包扔上去做一个数据收集看看。

```Docker

FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
WORKDIR /app

# copy csproj and restore as distinct layers
COPY src/*.csproj ./
RUN dotnet restore

COPY . ./

# copy everything else and build
RUN dotnet publish src -c Release -o /app/out


# build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1

# debug
# Install .NET Core SDK
RUN dotnet_sdk_version=3.1.100 \
    && curl -SL --output dotnet.tar.gz https://dotnetcli.azureedge.net/dotnet/Sdk/$dotnet_sdk_version/dotnet-sdk-$dotnet_sdk_version-linux-x64.tar.gz \
    && dotnet_sha512='5217ae1441089a71103694be8dd5bb3437680f00e263ad28317665d819a92338a27466e7d7a2b1f6b74367dd314128db345fa8fff6e90d0c966dea7a9a43bd21' \
    && echo "$dotnet_sha512 dotnet.tar.gz" | sha512sum -c - \
    && rm -rf /usr/share/dotnet \
    && rm -rf /usr/bin/dotnet \
    && mkdir -p /usr/share/dotnet \
    && tar -ozxf dotnet.tar.gz -C /usr/share/dotnet \
    && rm dotnet.tar.gz \
    && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet \
    # Trigger first run experience by running arbitrary cmd
    && dotnet help
RUN dotnet tool install --global dotnet-trace
RUN dotnet tool install -g dotnet-dump
RUN dotnet tool install --global dotnet-counters
ENV PATH="$PATH:/root/.dotnet/tools"

# end debug

WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "Your-APP.dll"]


```

---

更新发布，等待服务正常启动之后，

使用 ab -c 300 -n 3000 'http://172.16.2.52/v1/user/test/account-balance' 模拟 300 个用户同时请求，

使得程序进入上述的“假死状态”。

接着立即进入容器，执行'dotnet-trace collect -p 1' 开始收集日志。

最后拿到了一份大概 13M trace.nettrace 数据, 这个文件是 PerView 支持的格式，

在 MacOS 或者 Linux 上无法使用。

好在 dotnet-trace convert 可以将 trace.nettrace 转换成 speedscope/chromium 两种格式。

#### speedscope/chromium

- [speedscope:A fast, interactive web-based viewer for performance profiles.](https://github.com/jlfwong/speedscope)

- [chrome-devtools evaluate-performance ](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/)

- [全新 Chrome Devtool Performance 使用指南](https://zhuanlan.zhihu.com/p/29879682)

```sh
$dotnet-trace convert 20200606-1753-trace.nettrace.txt  --format Speedscope
$dotnet-trace convert 20200606-1753-trace.nettrace.txt --format chromium
$speedscope 20200606-1753-trace.nettrace.speedscope.json
```

然后，炸鸡了。

```log
➜  Downloads speedscope 20200606-1625.trace.speedscope.json
Error: Cannot create a string longer than 0x3fffffe7 characters
    at Object.slice (buffer.js:652:37)
    at Buffer.toString (buffer.js:800:14)
    at main (/home/liguobao/.nvm/versions/node/v12.16.2/lib/node_modules/speedscope/bin/cli.js:69:39)
    at processTicksAndRejections (internal/process/task_queues.js:97:5)

Usage: speedscope [filepath]

If invoked with no arguments, will open a local copy of speedscope in your default browser.
Once open, you can browse for a profile to import.

If - is used as the filepath, will read from stdin instead.

  cat /path/to/profile | speedscope -
```

哦， Buffer.toString 炸鸡了。

看一眼 20200606-1625.trace.speedscope.json 多大?

900M。

牛逼。

那换 Chrome performance 看看。

![chrome-performance](http://qiniu.house2048.cn/chrome-performance.png)

手动装载一下 20200606-1753-trace.nettrace.chromium.json 看看。

等下，20200606-1753-trace.nettrace.chromium.json 这货多大？

哦，4G。应该没事，Intel NUC 主机内存空闲 20G，吃掉 4G 完全是没有问题的。

![chrome_performance_load_json](http://qiniu.house2048.cn/chrome_performance_load_json.png)

看着进度条加载，看着内存涨起来，

然后...Chrome 控制台奔溃。再见再见，原来大家彼此完全没有信任了。

唉，再来一次，把文件控制在 5M 左右看看。

最后，把 20200606-1753-trace.nettrace.chromium.json 控制在 1.5G 了，

终于可以正常加载起来了。

---

### Chrome Performance

首先， 我们看到监控里面有一堆的线程

![Chrome%20performance_threading_74](http://qiniu.house2048.cn/Chrome%20performance_threading_74.png)

随便选一个线程看看它做撒,选择 Call Tree 之后 点点点点点。

![20performance_74_call_tree](http://qiniu.house2048.cn/Chrome%20performance_74_call_tree.png)

从调用栈能看到 整个线程当前状态是“PerformWaitCallback”

整个操作应该的开头应该是

Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.KestrelConnection.System.Threading.IThreadPoolWorkItem.Execute()

PS： Kestrel (https://github.com/aspnet/KestrelHttpServer) is a lightweight cross-platform web server that supports .NET Core and runs on multiple platforms such as Linux, macOS, and Windows. Kestrel is fully written in .Net core. It is based on libuv which is a multi-platform asynchronous eventing library.

PS 人话： .NET Core 内置的 HTTP Server，和 Spring Boot 中的 tomcat 组件类似

很正常，说明请求正常到了我们的服务里面了。

再看一下剩下的调用链信息。

![dotnet-call-tree-di](http://qiniu.house2048.cn/dotnet-call-tree-di.png)

简要的调用信息日志在这里：

```

System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.KestrelConnection+<ExecuteAsync>d__32.MoveNext()
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.HttpConnectionMiddleware`1[System.__Canon].OnConnectionAsync(class Microsoft.AspNetCore.Connections.ConnectionContext)   Microsoft.AspNetCore.Server.Kestrel.Core.Internal.HttpConnection.ProcessRequestsAsync(class Microsoft.AspNetCore.Hosting.Server.IHttpApplication`1<!!0>)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)
System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.HttpConnection+<ProcessRequestsAsync>d__12`1[System.__Canon].MoveNext()
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.HttpProtocol.ProcessRequestsAsync(class Microsoft.AspNetCore.Hosting.Server.IHttpApplication`1<!!0>)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)
System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.HttpProtocol+<ProcessRequestsAsync>d__216`1[System.__Canon].MoveNext()
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.HttpProtocol.ProcessRequests(class Microsoft.AspNetCore.Hosting.Server.IHttpApplication`1<!!0>)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)
System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)
Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.HttpProtocol+<ProcessRequests>d__217`1[System.__Canon].MoveNext()
Microsoft.AspNetCore.Hosting.HostingApplication.ProcessRequestAsync(class Context)
Microsoft.AspNetCore.HostFiltering.HostFilteringMiddleware.Invoke(class Microsoft.AspNetCore.Http.HttpContext)
UserCenter.ErrorHandlingMiddleware.Invoke(class Microsoft.AspNetCore.Http.HttpContext)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)
System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)

......
......
......

e.StaticFiles.StaticFileMiddleware.Invoke(class Microsoft.AspNetCore.Http.HttpContext)
Microsoft.AspNetCore.Builder.UseMiddlewareExtensions+<>c__DisplayClass4_1.<UseMiddleware>b__2(class Microsoft.AspNetCore.Http.HttpContext)
dynamicClass.lambda_method(pMT: 00007FB6D3BBE570,class System.Object,pMT: 00007FB6D4739560,pMT: 00007FB6D0BF4F98)
Microsoft.AspNetCore.Builder.UseMiddlewareExtensions.GetService(class System.IServiceProvider,class System.Type,class System.Type)
Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceProviderEngineScope.GetService(class System.Type)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)

Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor`2[Microsoft.Extensions.DependencyInjection.ServiceLookup.RuntimeResolverContext,System.__Canon].VisitCallSite(class Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceCallSite,!0)
System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(!!0&)
System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(!!0&)

```

看到这里，其实又有了一些很给力的信息被暴露出来了。

---

#### PerformWaitCallback

- 直接字面理解，线程正在执行等待回调

#### 调用链信息

耐心点，把所有的电调用链都展开。

我们能看到程序已经依次经过了下面几个流程：

->ProcessRequestsAsync(系统)

->ErrorHandlingMiddleware(已经加载了自定义的错误中间件)

-> HostFilteringMiddleware(加载了 Filter 中间件)

-> Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor(调用链中的最后一个操作)

对应最上面的日志来说，

请求进来，经过了中间件和 Filter 都是没问题的，

最后在 DependencyInjection（依赖注入） 中没有了踪迹。

到这里，

再次验证我们昨天的思路：

### 这是一个 “资源阻塞问题”产生的问题

虽然做 ptrace 是想能直接抓到“凶手”的，

最后发现并没有能跟踪到具体的实现，

那可咋办呢？

## 控制变量实践

已知：

- 并发 300 访问 /v1/user/test/account-balance 接口程序会假死
- 移除 Filter 中间件不能解决问题
- 并发 300 访问 /v1/health 健康检查接口程序正常
- ptrace 信息告诉我们有“东西”在阻塞 DI 容器创建某些实例

开始控制变量 + 人肉二分查找。

### 挪接口代码

/v1/user/test/account-balance 的逻辑是由 AccountService 实现的，

AccountService 大概依赖四五个其他的 Service 和 DBContext，

最主要的逻辑是加载用户几个账号，然后计算一下 balance。

大概代码如下：

```c#
/// <summary>
/// 获取用户的AccountBalance汇总信息
/// </summary>
public async Task<AccountBalanceStat> LoadAccountBalanceStatAsync(string owner)
{
    // 数据库查询
    var accounts = await _dbContext.BabelAccounts.Where(ac => ac.Owner == owner).ToListAsync();
    // 内存计算
    return ConvertToAccountBalanceStat(accounts);
}
```

什么都不改，直接把代码 CP 到 Health 接口测一下。

神奇，300 并发抗住了。

结论：

- 上面这一段代码并不会导致服务僵死
- 数据库驱动没有问题，DbContext 没有问题，数据库资源使用没有问题
- 当前并不会触发 DI 容器异常， 问题出在 /v1/user/test/account-balance 初始化

### account-balance 有什么神奇的东西吗？

```c#

/// <summary>
/// 查询用户的Brick账号余额
/// </summary>
[HttpGet("v1/user/{owner}/account-balance")]
[SwaggerResponse(200, "获取成功", typeof(AccountBrickStat))]
public async Task<IActionResult> GetAccountBricks(
    [FromRoute, SwaggerParameter("所有者", Required = true)] string owner)
{
    owner = await _userService.FindOwnerAsync(owner);
    return Ok(new { data = await _accountService.LoadAccountAsync(owner), code = 0 });
}

```

我们刚刚验证了 LoadAccountAsync 的代码是没有问题的，

要不 UserService DI 有问题，要不 AccountService DI 有问题。

把 UserService 加入到 HealthController 中。

```c#

public HealthController(UserService userService, UserPoolDataContext dbContext)
{
    _dbContext = dbContext;
    _userService= userService;
}
```

Bool。

300 并发没有撑住，程序僵死啦。

完美，

问题应该在 UserService DI 初始化了。

接下来就是一个个验证 UserService DI 需要的资源，

EmailSDK 没有问题，

HTTPHeaderTools 没有问题，

UserActivityLogService 没有问题。

RedisClient...
RedisClient...
RedisClient...

OK
OK
Ok

复现炸鸡了。

---

## 原来是 Redis 的锅？

是，

也不是。

先看下我们 RedisClient 是怎么使用的。

```C#
// startup.cs 注入了单例的ConnectionMultiplexer
// 程序启动的时候会调用InitRedis
private void InitRedis(IServiceCollection services)
{
    services.AddSingleton<ConnectionMultiplexer, ConnectionMultiplexer>(factory =>
    {
        ConfigurationOptions options = ConfigurationOptions.Parse(Configuration["RedisConnectionString"]);
        options.SyncTimeout = 10 * 10000;
        return ConnectionMultiplexer.Connect(options);
    });
}

//RedisClient.cs 通过构造函数传入

public class RedisClient
{
    private readonly ConnectionMultiplexer _redisMultiplexer;

    private readonly ILogger<RedisClient> _logger;

    public RedisClient(ConnectionMultiplexer redisMultiplexer, ILogger<RedisService> logger)
    {
        _redisMultiplexer = redisMultiplexer;
        _logger = logger;
    }
}
```

DI 初始化 RedisClient 实例的时候，

需要执行 ConnectionMultiplexer.Connect 方法，

ConnectionMultiplexer.Connect 是同步阻塞的。

ConnectionMultiplexer.Connect 是同步阻塞的。

ConnectionMultiplexer.Connect 是同步阻塞的。

一切都能解释了。

怎么改？

```c#
// InitRedis 直接把链接创建好，然后直接注入到IServiceCollection中
private void InitRedis(IServiceCollection services)
{
    ConfigurationOptions options = ConfigurationOptions.Parse(Configuration["RedisConnectionString"]);
    options.SyncTimeout = 10 * 10000;
    var redisConnectionMultiplexer = ConnectionMultiplexer.Connect(options);
    services.AddSingleton(redisConnectionMultiplexer);
    Log.Information("InitRedis success.");
}

```

发布验证，

开门放并发 300 + 3000 请求。

完美抗住，丝一般顺滑。

### 还有更优的写法吗？

- 看了下微软 Cache 中间件源码，更好的做法应该是通过信号量+异步锁来创建 Redis 链接，下来再研究一下
- 数据库中可能也存在类似的问题，不过当前会在 Startup 中戳一下数据库连接，应该问题不大。

## 复盘

程序启动的时候依赖注入容器同步初始化 Redis 可能很慢(几秒甚至更长)的时候，

其他的资源都在同步等待它初始化好，

最终导致请求堆积，引起程序雪崩效应。

Redis 初始化过慢并不每次都发生， 所以之前服务也只是偶发。

DI 初始化 Redis 连接的时候，redis 打来连接还是个同步的方法，

这种情况下还可能发生异步请求中等待同步资源产生阻塞问题。

同时还需要排查使用其他外部资源的时候是否会触发同类问题。

---

## 几个通用的小技巧

- ptrace 对此类问题分析很有意义，不同语言框架都有类似的实现

- 同步、异步概念的原理和实现都要了解，这个有利于理解一些奇奇怪怪的问题

- 火焰图、Chrome dev Performance 、speedscope 都是好东西

- Debug 日志能给更多的信息，在隔离生产的情况下大胆使用

- 这辈子都不可能看源码的，写写 CURD 多美丽？源码真香，源码真牛逼。

- 控制变量验证，大胆假设，小心求证，人肉二分查，先怀疑自己再怀疑框架

- 搞事的时候不要自己一个人，有 Bug 一定要拉上小伙伴一起吃

---

## 相关资料

- [IBM Developer ptrace 嵌入式系统中进程间通信的监视方法](https://www.ibm.com/developerworks/cn/linux/l-cn-embedded-ptrace/index.html)

- [分析进程调用 pstack 和 starce](https://www.jianshu.com/p/d6686cb72f68)

- [pstack 显示每个进程的栈跟踪](https://wangchujiang.com/linux-command/c/pstack.html)

- [微软：dotnet-trace performance analysis utility](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)

- [知乎：全新 Chrome Devtool Performance 使用指南](https://zhuanlan.zhihu.com/p/29879682)

- [speedscope A fast, interactive web-based viewer for performance profiles.](https://github.com/jlfwong/speedscope)

- [jdk 工具之 jstack(Java Stack Trace)](https://www.cnblogs.com/duanxz/p/5487576.html)

- [阮一峰:如何读懂火焰图？](https://ruanyifeng.com/blog/2017/09/flame-graph.html)
