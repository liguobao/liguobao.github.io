---
layout: post
title: 每周开源项目分享-年轻人的第一个APM之Skywalking
category: 每周开源项目分享
date: 2018-09-21
tags:
- Github
- Open Source
- skywalking
- 每周开源项目分享
---

# 年轻人的第一个APM之Skywalking

## 前言

什么是APM?全称:Application Performance Management

可以参考这里:

```text
现代APM体系，基本都是参考Google的Dapper（大规模分布式系统的跟踪系统）的体系来做的。通过跟踪请求的处理过程，来对应用系统在前后端处理、服务端调用的性能消耗进行跟踪，关于Dapper的介绍可以看这个链接：Dapper，大规模分布式系统的跟踪系统 by bigbully

作者：刀把五
链接：https://www.zhihu.com/question/27994350/answer/118821214
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

最早使用APM还是在携程里面搬砖的时候,当时使用的是大宗点评网开源的[dianping/cat](https://github.com/dianping/cat)框架.

后来到了新公司,因为历史包袱有点多,追踪性能问题太麻烦,用过收费的[New Relic | Real-time insights for modern software](https://newrelic.com/) ,newrelic按照CPU核数和内存来收费,实在太贵了我们就放弃了.

再后来我们调研一下市面的其他方案,看到了这个知乎讨论给了不少的东西.

[有什么知名的开源apm(Application Performance Management)工具吗？](https://www.zhihu.com/question/27994350)

当时看到[naver/pinpoint](https://github.com/naver/pinpoint) 和[apache/incubator-skywalking](https://github.com/apache/incubator-skywalking) 都很不错.

一个是韩国搜索团队开源的,一个是国内个人用户开源,已经到了apache孵化器了.

于是两个都试用了一下, 最后由于那时候马上考虑上分表分库组件 sharding-jdbc-dangdang, skywalking也要对应的支持,所以决定用skywalking试试.

再后来又跑路了,不好意思给那边留下坑就没继续搭建看. 到了新公司PHP/Python/Java什么都写,开始两三个月也没管这个.

最近不是太忙了,新公司这边服务端API暂时被我带成了dotnet core技术栈,233...

同时发现当前用的EF框架偶尔会因为不小心就写出了性能很差的SQL,测试环境基本看不出来,到了生产可能就炸.

前阵子看到dalao [倾竹](https://www.zhihu.com/people/liuhaoyang) 把dotnet core agent写出来了, 于是爽歪歪就开始gang了.

## 开始搭建skywalking

github:[incubator-skywalking](https://github.com/apache/incubator-skywalking)

当前release版本为5.0RC2,最新版本6.X正在开发中.

所以当前我这里是基于5.0 RC2来搭建的.

官方向导方案在这里:[incubator-skywalking/blob/5.x/docs/README.md](https://github.com/apache/incubator-skywalking/blob/5.x/docs/README.md)

中文文档在这里:[incubator-skywalking/blob/5.x/docs/README_ZH.md](https://github.com/apache/incubator-skywalking/blob/5.x/docs/README_ZH.md)

我这里今天还是全程docker部署.

以下操作来自[JaredTan95/skywalking-docker](https://github.com/JaredTan95/skywalking-docker) dalao准备的docker部署.

预备条件:

- docker

- elasticsearch

### 启动Elasticsearch

```sh

# Elasticsearch版本要求5.x

docker run -p 9200:9200 -p 9300:9300 -e cluster.name=elasticsearch -e xpack.security.enabled=false --name=elasticsearch --restart=always -d wutang/elasticsearch-shanghai-zone

```

启动好了访问一下 [http://localhost:9200](http://localhost:9200) 看看,看到一下的内容说明ES已经正常启动了.

```json
{
    "name": "_PNUyiW",
    "cluster_name": "elasticsearch",
    "cluster_uuid": "",
    "version": {
        "number": "5.6.10",
        "build_hash": "b727a60",
        "build_date": "2018-06-06T15:48:34.860Z",
        "build_snapshot": false,
        "lucene_version": "6.6.1"
    },
    "tagline": "You Know, for Search"
}
```

接着使用 docker inspect elasticsearch |grep IPAddress 查看一下 elasticsearch 当前IP.

```log
➜  ✗ docker inspect elasticsearch |grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.27.0.2",
```

## 启动 Skywalking UI + Skywalking collector

dalao wutang的[wutang/skywalking-docker](https://hub.docker.com/r/wutang/skywalking-docker/)已经把UI和collector打包到一个镜像里面了,完全可以独立安装.

所以我这里采用的也是这个方案.

```sh
docker run -p 8080:8080 -p 10800:10800 -p 11800:11800 -p 12800:12800 -e ES_CLUSTER_NAME=elasticsearch -e ES_ADDRESSES=上一步拿到的elasticsearchIP:9300 -d wutang/skywalking-docker:5.x

```

启动好了之后打开 [localhost:8080](localhost:8080),如果UI页面没有500/404错误,说明整个系统已经正常启动了.

PS:默认账号密码是:admin admin,可以在docker run指定 UI_ADMIN_PASSWORD环境变量自定义密码.

![UI](https://wx4.sinaimg.cn/mw1024/64d1e863gy1fvh3iegq92j21kn0st41c.jpg)

如果有错误的话,大概率是ES没有连上,检查一下ES是不是还活着,再不行就进到容器里面看日志.日志默认路径:/apache-skywalking-apm-incubating/logs

### Agent接入

当前已经有Java/C#(dotnet core)/Node.js的Agent了.

对应的话Java支持是最多的,其他两个我看下来基本就是主流比较多的一些框架都基本有了.

对应agent框架链接:

- dotnet core: [OpenSkywalking/skywalking-netcore](https://github.com/OpenSkywalking/skywalking-netcore)

- node.js:[OpenSkywalking/skywalking-nodejs](https://github.com/OpenSkywalking/skywalking-nodejs)

理论上应该遵循[http://opentracing.io/](http://opentracing.io/) API标准的.

Java agent 主仓库就有,直接去看release即可.

今天我们肯定是用dotnet core 啦.

dotnet core当前支持的库和中间件有下面这些:

- [ASP.NET Core](https://github.com/aspnet)
- [.NET Core BCL types (HttpClient and SqlClient)](https://github.com/dotnet/corefx) 
- [EntityFrameworkCore](https://github.com/aspnet/EntityFrameworkCore)
- [Npgsql.EntityFrameworkCore.PostgreSQL](https://github.com/npgsql/Npgsql.EntityFrameworkCore.PostgreSQL)
- [Pomelo.EntityFrameworkCore.MySql](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql)
- [CAP](https://github.com/dotnetcore/CAP)

嗯,该有的都有了.

先引入一下[SkyWalking.AspNetCore](https://www.nuget.org/packages/SkyWalking.AspNetCore/)的Package.

```sh
dotnet add package SkyWalking.AspNetCore --version 0.3.0
```

酌情新增 SkyWalking.Diagnostics.EntityFrameworkCore, SkyWalking.Diagnostics.HttpClient, SkyWalking.Diagnostics.EntityFrameworkCore.Npgsql,SkyWalking.Diagnostics.EntityFrameworkCore.Pomelo.MySql 等等...

或者直接在xxx.csproj 新增下面这些包.

```xml
    <PackageReference Include="SkyWalking.AspNetCore" Version="0.3.0"/>
    <PackageReference Include="SkyWalking.Diagnostics.EntityFrameworkCore" Version="0.3.0"/>
    <PackageReference Include="SkyWalking.Diagnostics.HttpClient" Version="0.3.0"/>
    <PackageReference Include="SkyWalking.Diagnostics.EntityFrameworkCore.Npgsql" Version="0.3.0"/>
    <PackageReference Include="SkyWalking.Diagnostics.EntityFrameworkCore.Pomelo.MySql" Version="0.3.0"/>
```

然后在 Startup.cs的ConfigureServices 方法中添加引用

```C#
// using SkyWalking.AspNetCore;
// using SkyWalking.Diagnostics.EntityFrameworkCore;
// using SkyWalking.Diagnostics.HttpClient;
// using SkyWalking.Diagnostics.SqlClient;

 services.AddSkyWalking(option =>
            {
                option.ApplicationCode = "my-first-api";
                option.DirectServers = "127.0.0.1:11800";
                // 每三秒采样的Trace数量,-1 为全部采集
                option.SamplePer3Secs = -1;
            }).AddEntityFrameworkCore(c => { c.AddPomeloMysql(); })
            .AddHttpClient();
```

接着启动应用.

看到有类似的日志输入,说明已经应用已经正常连接到SkyWalking了.

```log

info: SkyWalking.Remote.GrpcApplicationService[0]
      Register application instance success. [applicationInstanceId] = 31
SkyWalking.Remote.GrpcApplicationService:Information: Register application instance success. [applicationInstanceId] = 31
info: SkyWalking.Remote.GrpcApplicationService[0]
      Register application instance success. [applicationInstanceId] = 31
```

这时候我们打开[http://localhost:8080/#/monitor/dashboard](http://localhost:8080/#/monitor/dashboard)

![2](https://wx1.sinaimg.cn/mw1024/64d1e863gy1fvh43zghu1j21jp0jugnp.jpg)

可以看到APP已经有数量了.

接着我们访问一下已有的API/Web页面,就能看到对应的信息了.

![3](https://wx2.sinaimg.cn/mw1024/64d1e863gy1fvh4ctdxzyj21jp0nzjuc.jpg)

点一下对应的URL.

![4](https://wx1.sinaimg.cn/mw1024/64d1e863gy1fvh4qdgglvj21kf0s377r.jpg)

http client请求(其实是查询ES):

![5](https://wx1.sinaimg.cn/mw1024/64d1e863gy1fvh4qdgglvj21kf0s377r.jpg)

Topology Map

![234](https://wx2.sinaimg.cn/mw1024/64d1e863gy1fvh4wixvq2j21iw0n0q4y.jpg)

其他的一些功能就看自己玩了.

本期结束...
