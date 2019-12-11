﻿# RabbitMQ.Client.Core.DependencyInjection

<a href="https://www.nuget.org/packages/RabbitMQ.Client.Core.DependencyInjection/" alt="NuGet package"><img src="https://img.shields.io/nuget/v/RabbitMQ.Client.Core.DependencyInjection.svg" /></a><br/>
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/f688764d2ba340099ec50b74726e25fd)](https://app.codacy.com/app/AntonyVorontsov/RabbitMQ.Client.Core.DependencyInjection?utm_source=github.com&utm_medium=referral&utm_content=AntonyVorontsov/RabbitMQ.Client.Core.DependencyInjection&utm_campaign=Badge_Grade_Dashboard)<br/>

This repository contains the library that provides functionality to wrap [RabbitMQ.Client](https://github.com/rabbitmq/rabbitmq-dotnet-client) code and register it via dependency injection mechanism.

## Usage

This section contains only example of basic usage of the library. You can find the [detailed documentation](./docs/changelog.md) files in the docs directory, all functionality covered here.
### Producer

To produce messages in the RabbitMQ queue you have to go through the routine of configuring RabbitMQ connection and exchanges. In your `Startup` file you can do it simply calling couple methods in fluent Api way.
```csharp
public static IConfiguration Configuration { get; set; }

public void ConfigureServices(IServiceCollection services)
{
    var rabbitMqSection = Configuration.GetSection("RabbitMq");
    var exchangeSection = Configuration.GetSection("RabbitMqExchange");
    
    services.AddRabbitMqClient(rabbitMqSection)
        .AddProductionExchange("exchange.name", exchangeSection);
}
```
By calling `AddRabbitMqClient` you add a singleton `IQueueService` that provides functionality of sending messages to queues. `AddProductionExchange` configures exchange to queues bindings (presented as json configuration) that allow messages routing properly. 
Example of `appsettings.json` is two sections below. You can also configure everything manually. For more information see the [docs](./docs/changelog.md).

Now you can inject an instance implementing `IQueueService` inside anything you want.

```csharp
[Route("api/[controller]")]
public class HomeController : Controller
{
    private readonly IQueueService _queueService;
    public HomeController(IQueueService queueService)
    {
        _queueService = queueService;
    }
}
```

Now you can send messages using `Send` or `SendAsync` methods.

```csharp
var messageObject = new
{
    Id = 1,
    Name = "RandomName"
};

queueService.Send(
    @object: messageObject,
    exchangeName: "exchange.name",
    routingKey: "routing.key");
```

You can also send messages with delay.
```csharp
queueService.Send(
    @object: messageObject,
    exchangeName: "exchange.name",
    routingKey: "routing.key",
    secondsDelay: 10);
```
 The mechanism of sending delayed messages also described in the documentation. Dive into it for more information.
 
### Consumer

After making message production possible let's make the consumption possible too! Imagine that consumer will be a simple console application.

```csharp
class Program
{
    const string ExchangeName = "exchange.name";
    public static IConfiguration Configuration { get; set; }
    
    static void Main()
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
        Configuration = builder.Build();
    
        var serviceCollection = new ServiceCollection();
        ConfigureServices(serviceCollection);
    
        var serviceProvider = serviceCollection.BuildServiceProvider();
        var queueService = serviceProvider.GetRequiredService<IQueueService>();
        queueService.StartConsuming();
    }
    
    static void ConfigureServices(IServiceCollection services)
    {
        var rabbitMqSection = Configuration.GetSection("RabbitMq");
        var exchangeSection = Configuration.GetSection("RabbitMqExchange");
    
        services.AddRabbitMqClient(rabbitMqSection)
            .AddConsumptionExchange("exchange.name", exchangeSection)
            .AddMessageHandlerSingleton<CustomMessageHandler>("routing.key");
    }
}
```

You have to configure everything almost the same way as you have already done with producer. The main differences are that you need to declare (configure) consumption exchange calling `AddConsumptionExchange` instead of production exchange. For detailed information about difference in exchange declarations you may want to see the documentation.
And the most important part is adding custom message handlers by implementing `IMessageHandler` interface and calling `AddMessageHandlerSingleton<T>` or `AddMessageHandlerTransient<T>` methods. `IMessageHandler` is a simple subscriber, which receives messages from a queue by selected routing key.
The very last step is to start "listening" (subscribing) by simply calling `StartConsuming` method of `IQueueService`. After that you will start getting messages, and you can handle them in any way you want.

Message handler example.
```csharp
public class CustomMessageHandler : IMessageHandler
{
    readonly ILogger<CustomMessageHandler> _logger;
    public CustomMessageHandler(ILogger<CustomMessageHandler> logger)
    {
        _logger = logger;
    }
    
    public void Handle(string message, string routingKey)
    {
        // Do whatever you want!
        _logger.LogInformation("Hello world");
    }
}
```
There are async and non-cyclic message handler types, which allow you to do additional stuff. For more information see the documentation.

You can also find example projects in the repository inside the [examples](./examples) directory.

### Configuration
 
 In both cases for producing and consuming messages configuration file is the same. `appsettings.json` consists of those sections: (1) settings to connect to the RabbitMQ server and (2) sections that configure exchanges and queue bindings. You can have multiple exchanges and one configuration section per exchange.
Exchange sections define how to bind queues and exchanges with each other using specified routing keys. You allowed to bind a queue to an exchange with more than one routing key, but if there are no routing keys in the queue section, then that queue will be bound to the exchange with its name.
```json
{
  "RabbitMq": {
    "HostName": "127.0.0.1",
    "Port": "5672",
    "UserName": "guest",
    "Password": "guest"
  },
  "RabbitMqExchange": {
    "Type": "direct",
    "Durable": true,
    "AutoDelete": false,
    "DeadLetterExchange": "default.dlx.exchange",
    "RequeueFailedMessages": true,
    "Queues": [
	  {
        "Name": "myqueue",
        "RoutingKeys": [ "routing.key" ]
      }
    ]
  }
}
```
For more information about `appsettings.json` file format and manual configuration see the documentation file.

## Versioning
For now this project uses semantic versioning that follows .Net Core versioning. That means that major and minor versions are equal .Net Core major and minor version but patch version will be used independently.

## Changelog

All notable changes being tracked in the [changelog](./docs/changelog.md) file.

## License
This library is licenced under GNU General Public License v3 that means you are free to use it anywhere you want but you have to provide to the community all modifying changes of the library. </br>
Also feel free to contribute!