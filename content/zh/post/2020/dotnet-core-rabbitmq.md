---
layout: post
title: .NET Core中使用RabbitMQ正确方式
category: dotnet core
date: 2018-01-15
tags:
- dotnet core
- 单元测试
- 测试工具
- ReportGenerator
- coverlet
- UnitTest
---

# .NET Core中使用RabbitMQ正确方式

首先甩官网:[http://www.rabbitmq.com/](http://www.rabbitmq.com/)

然后是.NET Client链接:[http://www.rabbitmq.com/dotnet.html](http://www.rabbitmq.com/dotnet.html)

GitHub仓库:[https://github.com/rabbitmq/rabbitmq-dotnet-client](https://github.com/rabbitmq/rabbitmq-dotnet-client)

下面直接进入正文,一共是两个主题:消费者怎么写?生产者怎么写?

## 消费者

在dotnet core mvc中,消费者肯定不能通过API或者其他的东西启动,理应是跟着程序一起启动的.

所以...

在dotnet core 2.0以上版本,我们直接用 IHostedService 接口实现.

- [.NET Core 中基于 IHostedService 实现后台定时任务](https://www.cnblogs.com/dudu/p/9647619.html)

- [Implementing background tasks in .NET Core 2.x webapps or microservices with IHostedService and the BackgroundService class](https://blogs.msdn.microsoft.com/cesardelatorre/2017/11/18/implementing-background-tasks-in-microservices-with-ihostedservice-and-the-backgroundservice-class-net-core-2-x/)


直接上代码.

```C#
// RabbitListener.cs 这个是基类,只实现注册RabbitMQ后到监听消息,然后每个消费者自己去重写RouteKey/QueueName/消息处理函数Process

using System;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;


namespace Test.Listener
{
    public class RabbitListener : IHostedService
    {

        private readonly IConnection connection;
        private readonly IModel channel;


        public RabbitListener(IOptions<AppConfiguration> options)
        {
            try
            {
                var factory = new ConnectionFactory()
                {
                    // 这是我这边的配置,自己改成自己用就好
                    HostName = options.Value.RabbitHost,
                    UserName = options.Value.RabbitUserName,
                    Password = options.Value.RabbitPassword,
                    Port = options.Value.RabbitPort,
                };
                this.connection = factory.CreateConnection();
                this.channel = connection.CreateModel();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"RabbitListener init error,ex:{ex.Message}");
            }
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            Register();
            return Task.CompletedTask;
        }





        protected string RouteKey;
        protected string QueueName;

        // 处理消息的方法
        public virtual bool Process(string message)
        {
            throw new NotImplementedException();
        }

        // 注册消费者监听在这里
        public void Register()
        {
            Console.WriteLine($"RabbitListener register,routeKey:{RouteKey}");
            channel.ExchangeDeclare(exchange: "message", type: "topic");
            channel.QueueDeclare(queue:QueueName, exclusive: false);
            channel.QueueBind(queue: QueueName,
                              exchange: "message",
                              routingKey: RouteKey);
            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                var result = Process(message);
                if (result)
                {
                    channel.BasicAck(ea.DeliveryTag, false);
                }
            };
            channel.BasicConsume(queue: QueueName, consumer: consumer);
        }

        public void DeRegister()
        {
            this.connection.Close();
        }


        public Task StopAsync(CancellationToken cancellationToken)
        {
            this.connection.Close();
            return Task.CompletedTask;
        }
    }

}



// 随便贴一个子类

using System;
using System.Text;
using Microsoft.Extensions.Options;
using Newtonsoft.Json.Linq;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

namespace Test.Listener
{
    public class ChapterLister : RabbitListener
    {


        private readonly ILogger<RabbitListener> _logger;

        // 因为Process函数是委托回调,直接将其他Service注入的话两者不在一个scope,
        // 这里要调用其他的Service实例只能用IServiceProvider CreateScope后获取实例对象
        private readonly IServiceProvider _services;

        public ChapterLister(IServiceProvider services, IOptions<AppConfiguration> options,
         ILogger<RabbitListener> logger) : base(options)
        {
            base.RouteKey = "done.task";
            base.QueueName = "lemonnovelapi.chapter";
            _logger = logger;
            _services = services;

        }

        public override bool Process(string message)
        {
            var taskMessage = JToken.Parse(message);
            if (taskMessage == null)
            {
                // 返回false 的时候回直接驳回此消息,表示处理不了
                return false;
            }
            try
            {
                using (var scope = _services.CreateScope())
                {
                    var xxxService = scope.ServiceProvider.GetRequiredService<XXXXService>();
                    return true;
                }

            }
            catch (Exception ex)
            {
                _logger.LogInformation($"Process fail,error:{ex.Message},stackTrace:{ex.StackTrace},message:{message}");
                _logger.LogError(-1, ex, "Process fail");
                return false;
            }

        }
    }
}

```

然后,记住....

注入到Startup.cs的时候,使用AddHostedService

```csharp
  services.AddHostedService<ChapterLister>();
```

消费者就这样玩了.

## 生产者咋玩呢?

这个其实更简单.

```csharp

using System;
using System.Net;
using Newtonsoft.Json.Linq;
using RestSharp;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using RabbitMQ.Client;
using Newtonsoft.Json;
using System.Text;

namespace Test.SDK
{
    public class RabbitMQClient
    {

        private readonly IModel _channel;

        private readonly ILogger _logger;




        public RabbitMQClient(IOptions<AppConfiguration> options, ILogger<RabbitMQClient> logger)
        {
            try
            {
                var factory = new ConnectionFactory()
                {
                    HostName = options.Value.RabbitHost,
                    UserName = options.Value.RabbitUserName,
                    Password = options.Value.RabbitPassword,
                    Port = options.Value.RabbitPort,
                };
                var connection = factory.CreateConnection();
                _channel = connection.CreateModel();
            }
            catch (Exception ex)
            {
                logger.LogError(-1, ex, "RabbitMQClient init fail");
            }
            _logger = logger;
        }

        public virtual void PushMessage(string routingKey, object message)
        {
            _logger.LogInformation($"PushMessage,routingKey:{routingKey}");
            _channel.QueueDeclare(queue: "message",
                                        durable: false,
                                        exclusive: false,
                                        autoDelete: false,
                                        arguments: null);
            string msgJson = JsonConvert.SerializeObject(message);
            var body = Encoding.UTF8.GetBytes(msgJson);
            _channel.BasicPublish(exchange: "message",
                                    routingKey: routingKey,
                                    basicProperties: null,
                                    body: body);
        }
    }
}
```

切记注入实例的时候用单例模式.

services.AddSingleton<RabbitMQClient, RabbitMQClient>();

全文完...

