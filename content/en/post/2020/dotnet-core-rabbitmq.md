---
layout: post
title: Using RabbitMQ in ASP.NET Core the Right Way
category: dotnet core
date: 2018-01-15
tags:
- dotnet core
- RabbitMQ
---

# Consumers and Producers in ASP.NET Core

Docs: http://www.rabbitmq.com/ — .NET client: http://www.rabbitmq.com/dotnet.html

## Consumer with IHostedService

Run consumers with the app lifecycle using `IHostedService`.

```csharp
public class RabbitListener : IHostedService
{
    private readonly IConnection _conn; private readonly IModel _channel;
    protected string RouteKey; protected string QueueName;

    public RabbitListener(IOptions<AppConfiguration> options)
    {
        var f = new ConnectionFactory {
            HostName = options.Value.RabbitHost,
            UserName = options.Value.RabbitUserName,
            Password = options.Value.RabbitPassword,
            Port = options.Value.RabbitPort,
        };
        _conn = f.CreateConnection();
        _channel = _conn.CreateModel();
    }

    public Task StartAsync(CancellationToken ct) { Register(); return Task.CompletedTask; }
    public Task StopAsync(CancellationToken ct) { _conn.Close(); return Task.CompletedTask; }

    public virtual bool Process(string message) => throw new NotImplementedException();

    void Register()
    {
        _channel.ExchangeDeclare(exchange: "message", type: "topic");
        _channel.QueueDeclare(queue: QueueName, exclusive: false);
        _channel.QueueBind(queue: QueueName, exchange: "message", routingKey: RouteKey);
        var consumer = new EventingBasicConsumer(_channel);
        consumer.Received += (m, ea) =>
        {
            var msg = Encoding.UTF8.GetString(ea.Body);
            if (Process(msg)) _channel.BasicAck(ea.DeliveryTag, false);
        };
        _channel.BasicConsume(queue: QueueName, consumer: consumer);
    }
}
```

Subclass and resolve scoped services via `IServiceProvider.CreateScope()` inside `Process`.

Register: `services.AddHostedService<YourListener>();`

## Producer

```csharp
public class RabbitMQClient
{
    private readonly IModel _channel; private readonly ILogger _logger;
    public RabbitMQClient(IOptions<AppConfiguration> opts, ILogger<RabbitMQClient> logger)
    {
        var f = new ConnectionFactory {
            HostName = opts.Value.RabbitHost,
            UserName = opts.Value.RabbitUserName,
            Password = opts.Value.RabbitPassword,
            Port = opts.Value.RabbitPort,
        };
        var conn = f.CreateConnection();
        _channel = conn.CreateModel();
        _logger = logger;
    }
    public void PushMessage(string routingKey, object message)
    {
        _logger.LogInformation($"PushMessage,routingKey:{routingKey}");
        _channel.QueueDeclare(queue: "message", durable: false, exclusive: false, autoDelete: false);
        var body = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(message));
        _channel.BasicPublish(exchange: "message", routingKey: routingKey, basicProperties: null, body: body);
    }
}
```

Register as singleton: `services.AddSingleton<RabbitMQClient>();`

