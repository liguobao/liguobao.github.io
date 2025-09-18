---
layout: post
title: .NET Core Web API Swagger 文档生成
category: linux
date: 2019-07-06
tags:
- dotnet core
- swagger
---
# .NET Core Web API Swagger 文档生成

REST API 中文档说明,用Swagger都快成了一种规范了,

之前在公司里面就折腾过了, 效果还是很不错的, 不过之前都是维护一个swagger json/yaml, 

后来发现其实可以直接在API实现的地方根据实现来生成swagger在线文档,

拖延症发作的我并没有去管, 这次有个新API在做, 于是折腾了一下.

## 起步

首先要有个.NET Core项目.

- [微软官方教程getting-started-with-swashbuckle](https://docs.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-2.2&tabs=netcore-cli)

- [Github/Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)

## 引入一下Swashbuckle.AspNetCore和Swashbuckle.AspNetCore.Annotations

```shell

# 主要的文档生成都在这里
dotnet add package Swashbuckle.AspNetCore --version 4.0.1

# 用来描述请求的相关信息
dotnet add package Swashbuckle.AspNetCore.Annotations --version 4.0.1

```

代码:

```csharp

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Zhihu404API.Dao;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using Swashbuckle.AspNetCore.Swagger;

namespace XX404API
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
            services.AddOptions().Configure<AppSettings>(Configuration);
            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new Info { Title = "XX-404 API", Version = "v1" });
                c.EnableAnnotations();
            });
        }



        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            // 启用Swagger
            app.UseSwagger();
            // 启动Swagger UI
            app.UseSwaggerUI(c =>
            {
                c.RoutePrefix = "docs";
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "Zhihu-404 API");
            });
            app.UseMvc();
        }
    }
}

```

然后随便扔两个Controller代码出来作为样例看看.

```csharp
// HealthController.cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;

namespace XX404API.Controllers
{
    [Route("api/v1/health")]
    [ApiController]
    public class HealthController : ControllerBase
    {
        // GET api/values
        [HttpGet]
        public ActionResult<IEnumerable<string>> Get()
        {
            return new string[] { "value1", "value2" };
        }
    }
}
```


```csharp
// HealthController.cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Swashbuckle.AspNetCore.Annotations;
using Newtonsoft.Json.Linq;
using RestSharp;

namespace XX404API.Controllers
{
    [ApiController]
    public class ExtendDataController : ControllerBase
    {

        public ExtendDataController()
        {

        }




        [HttpPost("api/v1/extend-data")]
        // 返回值格式会直接序列化这个typeof的类型
        [SwaggerResponse(200, "", typeof(int))]
        // SwaggerParameter会生成请求体的格式
        public ActionResult AddExtendData([FromBody, SwaggerParameter("原始数据")]List<DBExtendData> extendData)
        {
            if(extendData ==null || extendData.Any(a =>a.Id == default(long)))
            {
                throw new Exception("传入数据有误, 请检查每一项数据Id");
            }
            var count = 10;// _extendDataDapper.BulkInsert(extendData);
            return Ok(new { code = 0, data = count });
        }
    }

    class DBExtendData
    {
        [Column("id")]
        [JsonProperty(PropertyName = "id")]
        public long Id { get; set; }

        [Column("create_time")]
        [JsonProperty(PropertyName = "createTime")]
        public DateTime? CreateTime { get; set; }

        [Column("update_time")]
        [JsonProperty(PropertyName = "updateTime")]
        public DateTime? UpdateTime { get; set; } = DateTime.Now;
    }
}
```

然后访问 localhost:5000/docs 就能看到下面的文档了.

![](https://ws1.sinaimg.cn/large/64d1e863gy1g4q874s0x7j228o16owka.jpg)


![](https://ws1.sinaimg.cn/large/64d1e863gy1g4q87d3tlxj22ae0wqwhz.jpg)


好了, 全文完.

我去做饭了...



