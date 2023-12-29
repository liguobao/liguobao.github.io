---
layout: post
title: 每周开源项目分享-dotnet core 简易定时任务框架TimeJob
category: 每周开源项目分享
date: 2018-07-28
tags:
- Github
- Open Source
- dotnet core
---

# dotnet core 简易定时任务框架TimeJob

很多时候我们可能需要周期重复做一些事情, 定时任务框架应运而生.

在Linux下面crontab集合shell脚本做一些定时重复操作是常见通用的.

但是有时候我们可能需要在程序中做类似的事情,如:

1. 定时邮件推送

2. 定时监控日报生成

3. XXX...

Java这边,一般都使用Quartz框架简单实现定时任务.

.NET这边,也有Quartz.net,不过ASP.NET时代受制于IIS,经常会有同行小伙伴说抱怨定时任务偶尔突然就不跑.

参考文章:

1. [网站发布后在IIS上定时执行任务](https://blog.csdn.net/y1535623813/article/details/50537595)

2. [Quartz定时任务和IIS程序池闲置超时时间冲突解决方案](https://www.cnblogs.com/xielong/p/6802329.html)

到了dotnet core时代,自宿主不依赖IIS了,也有自己独立的主线程之后,我们做定时任务就很方便了.

开源dalao[Amamiya Yuuko](https://github.com/yukozh) 就自己撸了一个简易定时任务框架出来啦.

GitHub开源地址:[https://github.com/PomeloFoundation/dotNETCore-Extensions](https://github.com/PomeloFoundation/dotNETCore-Extensions)

Nuget地址:[Pomelo.AspNetCore.TimedJob](https://www.nuget.org/packages/Pomelo.AspNetCore.TimedJob/2.0.0-rtm-10046)

## TimeJob 使用教程

### Start.cs的ConfigureServices注入AddTimedJob服务

代码如下:

```C#
public void ConfigureServices(IServiceCollection services)
{
     services.AddTimedJob();
}
```

### Start.cs的Configure引入UseTimedJob中间件

```C#
 public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseTimedJob();
}
```

新建一个XXXJob.cs类,继承于Job

```C#

using Microsoft.Extensions.Options;
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.Linq;
using Pomelo.AspNetCore.TimedJob;

namespace Sample.Jobs
{
    public class TestJob : Job
    {
        public TestJob()
        {

        }

        [Invoke(Begin = "2018-07-27 00:00", Interval = 1000 * 600, SkipWhileExecuting = true)]
        public void Run()
        {
            Console.WriteLine(DateTime.Now.ToString()+",TestJob run...");
        }

    }

}

```

大功告成!

如果需要把定时任务相关的内容固化到数据库,可以参考:[Timed Job - Pomelo扩展包系列](http://www.1234.sh/post/pomelo-extensions-timed-jobs)

嗯?完了?...

对啊,结束了.

真结束了....