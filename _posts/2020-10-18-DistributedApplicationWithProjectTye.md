---
layout: post
title: "Distributed application with Project Tye"
categories: post
author: Rival Abdrakhmanov
date: 2020-10-18
---
In this post, I want to take a look at the new tool from the ASP team called Project Tye. It helps you to create and manage distributed applications locally. I'm going to show you an example of such applications.

![Title image](/images/2020-10-18-DistributedApplicationWithProjectTye/cover_distributed_app_with_project_tye.jpg)

# Application

We need to create a distributed application. It will consist of API and worker projects, and I'm going to connect them via RabbitMQ.

So, let's start with API. Create a new ASP.NET Core project.

```
$ dotnet new web --name Library.Api
```

Add a controller and a `BookDto`.

```c#
[ApiController]
[Route("api/[controller]")]
public class LibraryController : ControllerBase
{
    public LibraryController()
    {
    }

    [HttpPost]
    public async Task<IActionResult> Borrow(BookDto book)
    {
        return Ok();
    }
}
```

```c#
public class BookDto
{
    public string Title { get; set; }
    public string Author { get; set; }
    public string Content { get; set; }
}
```

For now, it's an empty controller. We'll add some logic later.

After that, create a new worker project.

```
$ dotnet new worker --name Library.Worker
```

Add to this project a library service.

```c#
public class LibraryService
{
    private readonly ILogger<LibraryService> _logger;

    public LibraryService(ILogger<LibraryService> logger)
    {
        _logger = logger;
    }

    public void Borrow(Book book)
    {
        _logger.LogInformation("Book {Title} by {Author}", book.Title, book.Author);
    }
}
```

```c#
public class Book
{
    public string Title { get; }
    public string Author { get; }
    public string Content { get; }

    public Book(string title, string author, string content)
    {
        Title = title;
        Author = author;
        Content = content;
    }
}
```

And register it in the DI system.

```c#
Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddSingleton<LibraryService>();
    });
```

# MassTransit

Next, we have to connect API service with workers.

[Asynchronous communication](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/asynchronous-message-based-communication) gives many advantages to distributed systems. Services become more autonomous and resilient to communication failures. Also, this type of communication increases your application performance and allows you to add new replicas of the service without any load balancer.

In my opinion, you need to avoid synchronous calls in the distributed system. If you have two services that communicate a lot in this way, you should think about combining them into one service.

Hence, we will use messaging.

There is an excellent library called [MassTransit](http://masstransit-project.com/), which simplifies message-based communication. With this library, you can use different message queues as transport. I'm going to show you an example with RabbitMQ.

Firstly, install the following nuget packages to both projects.

```
MassTransit.AspNetCore
MassTransit.RabbitMQ
```

Secondly, add MassTransit to the DI container. Modify the `Startup` class in the API project:

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddMassTransit(x =>
    {
        x.SetKebabCaseEndpointNameFormatter();
        x.UsingRabbitMq();
    });
    services.AddMassTransitHostedService();
}
```

And add some lines to the `Program` class in the Worker project.

```c#
Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services.AddSingleton<LibraryService>();
        services.AddMassTransit(x =>
        {
            x.SetKebabCaseEndpointNameFormatter();
            x.AddConsumers(Assembly.GetAssembly(typeof(Program)));
            x.UsingRabbitMq((context, cfg) =>
            {
                cfg.ConfigureEndpoints(context);
            });
        });
        services.AddMassTransitHostedService();
    });
```

We've configured MassTransit in our projects; now we're going to set up the interaction. Create a new project and add an interface that will be used as a message between services.

```
$ dotnet new classlib --name Library.Contracts
```

```c#
public interface BorrowBook   
{
    string Title { get; }
    string Author { get; }
    string Content { get; }
}
```

After that, we need to handle this message in the Worker project. Let's add a consumer.

```c#
public class BorrowBookConsumer : IConsumer<BorrowBook>
{
    private readonly LibraryService _libraryService;

    public BorrowBookConsumer(LibraryService libraryService)
    {
        _libraryService = libraryService;
    }

    public Task Consume(ConsumeContext<BorrowBook> context)
    {
        var msg = context.Message;
        var book = new Book(msg.Title, msg.Author, msg.Content);
        _libraryService.Borrow(book);
        return Task.CompletedTask;
    }
}
```

It catches the `BorrowBook` message and forwards the content from this message to the library service.

The last step is to send this message from the `LibraryController`. To do this, we have to add a send endpoint provider.

```c#
private readonly ISendEndpointProvider _sendEndpointProvider;
public LibraryController(ISendEndpointProvider sendEndpointProvider)
{
    _sendEndpointProvider = sendEndpointProvider;
}
```

Next, obtain an endpoint for our message and send it.

```c#
[HttpPost]
public async Task<IActionResult> Borrow(BookDto book)
{
    var endpoint = await _sendEndpointProvider.GetSendEndpoint(new Uri("exchange:borrow-book"));
    await endpoint.Send<BorrowBook>(new
    {
        book.Title,
        book.Author,
        book.Content
    });

    return Accepted();
}
```

Finally, our elementary distributed system is done, and we can test it. Do not forget that in addition to your services, you need to run a message queue. How to run all this together? Of course, you can use your IDE or just console, but I'm going to show you how to use [Project Tye](https://github.com/dotnet/tye) for this purpose.

# Project Tye

To start with, we have to install a package.

```
$ dotnet tool install -g Microsoft.Tye --version "0.4.0-alpha.20371.1"
```

Add a new `tye.yaml` file to the solution folder.

```yaml
name: service-kinds
services:
  - name: rabbitmq
    image: rabbitmq:3-management
    bindings:
      - name: rabbit
        port: 5672
        protocol: rabbitmq
      - name: management
        port: 15672

  - name: worker
    project: Library.Worker/Library.Worker.csproj
    replicas: 3

  - name: api
    project: Library.Api/Library.Api.csproj
    bindings:
      - port: 5000
```

And start with a command.

```
$ tye run
```

After that, your entire distributed system will start. You can find a beautiful dashboard at this address `http://127.0.0.1:8000/`.

![Tye services dashboard](/images/2020-10-18-DistributedApplicationWithProjectTye/tye-dashboard.png)

And four connections at the RabbitMQ dashboard `http://localhost:15672`.

![RabbitMQ dashboard](/images/2020-10-18-DistributedApplicationWithProjectTye/rabbit-dashboard.png)

Eventually, you can send a message and get the log.

```http
POST http://localhost:5000/api/library
Content-Type: application/json

{
  "Title": "Anna Karenina",
  "Author": "Leo Tolstoy",
  "Content": "Happy families are all alike; every unhappy family is unhappy in its own way."
}
```

![Message log](/images/2020-10-18-DistributedApplicationWithProjectTye/message-log.png)

## Competing consumers

[Competing consumers](https://docs.microsoft.com/en-us/azure/architecture/patterns/competing-consumers) is a pattern that increases the performance and availability of your application. If your application is under heavy load, you can run new consumers which will process messages from the queue in parallel. As you saw, we've already started three instances of our worker project. Let's send three requests and see what happens.

```http
POST http://localhost:5000/api/library
Content-Type: application/json

{
  "Title": "Anna Karenina",
  "Author": "Leo Tolstoy",
  "Content": "Happy families are all alike; every unhappy family is unhappy in its own way."
}

###

POST http://localhost:5000/api/library
Content-Type: application/json

{
  "Title": "The Bronze Horseman",
  "Author": "Alexander Pushkin",
  "Content": "A wave-swept shore, remote, forlorn: Here stood he, rapt in thought and drawn..."
}

###

POST http://localhost:5000/api/library
Content-Type: application/json

{
  "Title": "A Hero of Our Time",
  "Author": "Mikhail Lermontov",
  "Content": "My whole life has been merely a succession of miserable and unsuccessful denials of feelings or reason."
}

###
```

In logs, you'll see that different consumers will handle different messages. So, with MassTransit, you have this pattern for free.

```
[worker_cfa3748a-8]: Book Anna Karenina by Leo Tolstoy
...
[worker_aad6e09b-9]: Book The Bronze Horseman by Alexander Pushkin
...
[worker_767f01ef-9]: Book A Hero of Our Time by Mikhail Lermontov
```

## Docker

As you saw, tye can work with .NET projects and Docker images simultaneously. It has a lot more capabilities, like publishing to Kubernetes or using different extensions. I won't show all of them. One thing that I would like to pay attention to is the using of docker images. Tye allows you to build a docker image for the project or whole solution without any Dockerfiles. I think it's pretty cool. But I couldn't find a way to specify your own Dockerfile for a nonstandard one. Maybe, it'll be available in the future.

```
$ tye build ./Library.Api/Library.Api.csproj 
```

# MassTransit Platform

The last thing I want to discuss is this interesting MassTransit feature called [Platform](https://masstransit-project.com/platform/). If you create many such workers, it will be convenient for you to reuse some common parts.

Create a new project.

```
$ dotnet new classlib --name Library.OnPlatform
```

Install a nuget package `MassTransit.Platform.Abstractions`. Next, create a startup class:

```c#
public class LibraryStartup : IPlatformStartup
{
    public void ConfigureMassTransit(IServiceCollectionBusConfigurator configurator, IServiceCollection services)
    {
        services.AddSingleton<LibraryService>();
        configurator.AddConsumer<BorrowBookConsumer>();
    }

    public void ConfigureBus<TEndpointConfigurator>(IBusFactoryConfigurator<TEndpointConfigurator> configurator,
        IBusRegistrationContext context) where TEndpointConfigurator : IReceiveEndpointConfigurator
    {
    }
}
```

This class substitutes everything that you specify in the `Program.cs` class in the Worker project. It's like using WebHost in the ASP.NET Core applications. A lot of work is done under the hood; you only need to identify your consumers. Of course, you need to add a library service and a book class. After that, this project will be the same as the Worker project. If you want to build a docker image, you can use the following Dockerfile.

```docker
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
WORKDIR /src

COPY ./Library.OnPlatform/Library.OnPlatform.csproj ./Library.OnPlatform/
COPY ./Library.Contracts/Library.Contracts.csproj ./Library.Contracts/
RUN dotnet restore ./Library.OnPlatform/Library.OnPlatform.csproj 

COPY . .
RUN dotnet publish -c Release -o /app --no-restore ./Library.OnPlatform/Library.OnPlatform.csproj 

FROM masstransit/platform:7
WORKDIR /app
COPY --from=build /app ./
```

# Conclusion

In this post, we've looked at the distributed systems. I've shown you how the MassTransit library simplifies the messaging communications between your services. Also, we've created a straightforward example to demonstrate how new tool from ASP team can help you work with your distributed system locally. Both libraries have a lot more capabilities. If you want to know more about them, I'll put references bellow.

I hope it was interesting and fun.

Example you can find here:

[Link to GitHub Project](https://github.com/rafaelldi/DistributedLibrary)

# References
* https://github.com/dotnet/tye
* https://masstransit-project.com/
* https://masstransit-project.com/platform/
* https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture
* https://docs.microsoft.com/en-us/azure/architecture/patterns/competing-consumers

*Image: Photo by Regine Tholen on Unsplash*