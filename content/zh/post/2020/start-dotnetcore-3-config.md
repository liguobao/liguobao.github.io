---
layout: post
title: 手把手教你写dotnet core(读取配置文件)
category: dotnet core
date: 2018-07-01
tags:
- dotnet core
- docker
- asp.net core
---
# dotnet core(读取配置文件)

第一篇:[手把手教你写dotnet core(入门篇)](https://zhuanlan.zhihu.com/p/37460329)

第二篇:[手把手教你ASP.NET Core](https://zhuanlan.zhihu.com/p/37464924)

今天我们来学习怎么读取dotnet core程序的配置文件.

一般dotnet core配置文件都位于项目目录下,名为appsettings.json

## 直接读文件

```json
{
    "MySQLConnectionString": "server=mysql地址;port=端口号;database=数据库名字;uid=账号;pwd=密码;charset='utf-8';Allow User Variables=True;Connection Timeout=30;SslMode=None;",
    "RedisConnectionString": "redis数据库地址:端口,name=名字,keepAlive=1800,syncTimeout=10000,connectTimeout=360000,password=访问密码,ssl=False,abortConnect=False,responseTimeout=360000,defaultDatabase=1",
    "EmailAccount": "QQ邮箱账号",
    "EmailPassword": "QQ邮箱密码",
    "EmailSMTPDomain": "smtp.qq.com",
    "EmailSMTPPort": 587,
    "SenderAddress": "QQ邮箱账号",
    "ReceiverAddress": "QQ邮箱账号",
    "ReceiverName": "liguobao-test",
    "EncryptionConfigCIV": "加密向量,16个16进制数字",
    "EncryptionConfigCKEY": "加密秘钥,16个16进制数字",
    "ESURL":"http://127.0.0.1:9201/",
    "ESUserName":"",
    "ESPassword":"",
    "QQAPPID":"",
    "QQAPPKey":"",
    "QQAuthReturnURL":""
}

```

那么我们直接去读json然后序列化成对象是不是就可以了.

没毛病,确实是可以这样玩的. 代码大概是这样的...

```C#

        public class AppSettings
        {
            public string MySQLConnectionString {get;set;}

            public string RedisConnectionString {get;set;}

            // ....
        }

        public AppSettings JsonHelper(string jsonFilePath)
        {
            _movieJsonFilePath = jsonFilePath;
            if (!File.Exists(jsonFilePath))
            {
                var pvFile = File.Create(jsonFilePath);
                pvFile.Flush();
                pvFile.Dispose();
                return;

            }
            using (var stream = new FileStream(jsonFilePath, FileMode.OpenOrCreate))
            {
                try
                {
                    StreamReader sr = new StreamReader(stream);
                    JsonSerializer serializer = new JsonSerializer
                    {
                        NullValueHandling = NullValueHandling.Ignore,
                        Converters = { new JavaScriptDateTimeConverter() }
                    };
                    //构建Json.net的读取流  
                    using (var reader = new JsonTextReader(sr))
                    {
                        return serializer.Deserialize<AppSettings>(reader);
                    }

                }
                catch (Exception ex)
                {
                    return null;
                }
            }
        }
```

直接读文件然后反序列化实在有点麻烦,有没有简单点的办法啊.

客官,只要给钱什么都有.

## 使用ConfigurationBuilder 读取

直接上代码:

```C#
        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            var mySQLConnectionString = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json").Build()["MySQLConnectionString"];

            // ConnectionStrings节点下的MySQLConnectionString
            // var mySQLConnectionString = new ConfigurationBuilder()
            //         .SetBasePath(Directory.GetCurrentDirectory())
            //     .AddJsonFile("appsettings.json").Build()["ConnectionStrings:MySQLConnectionString"];
        }

```

嗯,就是这么简单粗暴,什么都不管...

辣鸡,还有没有更优雅点的方法啊.

哦,这样的要求么?那么我们上DI(自动注入)吧.

## DI读取配置文件

```C#
        //Startup.cs
        public IConfiguration Configuration { get; }

        public Startup(IHostingEnvironment env)
        {
            //够赞
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
            Configuration = builder.Build();
        }

         public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc();
            //将Configuration注入到APPConfiguration实例中
            services.AddOptions().Configure<AppSettings>(Configuration);
        }
```

然后我们在Controller中使用构造函数注入的方式获取APPConfiguration实例

```C#
        private AppSettings configuration;

        public AccountController(IOptions<AppSettings> configurationOption)
        {
            this.configuration = configurationOption.Value;
        }
```

然后就可以愉快的使用了.

本文结束...

拜...
