---
title: "gRPC with ASP.NET Core"
excerpt: "In this post, we continue our talk about communication between services. Today we’re going to create gRPC server and a client for it. Also, we’ll talk about the differences between gRPC and REST."
header:
  og_image: /images/2020-01-29-grpc-with-asp-net-core/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2020-01-29
tags: ["Connection", "ASP.NET Core", "gRPC", "Protobuf"]
---

![Title image](/images/2020-01-29-grpc-with-asp-net-core/cover.jpg)

# gRPC
[gRPC](https://grpc.io/) (gRPC Remote Procedure Call) is a modern RPC framework. It works over [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2) only and uses [proto format](https://developers.google.com/protocol-buffers/docs/overview) by default for describing an interface and messages. gRPC is a contract-first approach, so you need to create a .proto file and generate your server and client from it. All modern languages support gRPC so that you can use it in a multilanguage environment.

The best part here is that you can add gRPC to your ASP.NET Core application in a very natural way, and everything will be the same (logging, DI, authentication). I show you in the next parts how to deal with it.

# Server
Now, let’s build a server. We’ll create the same blog service, as we did in the [REST post](https://rafaelldi.blog/rest-api-with-asp-net-core/), so that we can compare these two solutions.

```protobuf
syntax = "proto3";

service Blog {
  rpc Create (Article) returns (Empty);
  rpc List (Empty) returns (ListOfArticles);
}

message Article {
  string name = 1;
  string author = 2;
  string content = 3;
}

message ListOfArticles {
  repeated Article articles = 1;
}

message Empty {
}
```

And there is a contract for REST service from the previous project.

```yaml
openapi: 3.0.0
info:
  title: Blog API
  description: API for working with blog
  version: 1.0.0
servers:
  - url: http://localhost:5000
    description: Localhost server
paths:
  /api/article/create:
    post:
      summary: Create a new article in a blog.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/definitions/Article"
      responses:
        '200':
          description: Create
  /api/article/list:
    get:
      summary: Return a list of articles.
      responses:
        '200':
          description: A JSON array of articles
          content:
            application/json:
              schema:
                $ref: "#/definitions/Articles"
definitions:
  Article:
    type: object
    properties:
      title:
        type: string
      author:
        type: string
      content:
        type: string
  Articles:
    type: array
    items:
      $ref: "#/definitions/Article"
```

You can see that the proto contract is shorter than OpenAPI.

After that, create a default grpc project with the following command. You can check out an example in [my GitHub repo](https://github.com/rafaelldi/GrpcWithAspNetCore).

```
dotnet new grpc
```

I removed default service and proto file, add my `Blog.proto` and generate service with a build command.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <Protobuf Include="..\Blog.proto" GrpcServices="Blog" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.24.0" />
  </ItemGroup>

</Project>
```

```c#
public class BlogService : Blog.BlogBase
{
    private readonly ArticleService _articleService;

    public BlogService(ArticleService articleService)
    {
        _articleService = articleService;
    }

    public override Task<Empty> Create(Article article, ServerCallContext context)
    {
        _articleService.Articles.Add(article);
        return Task.FromResult(new Empty());
    }

    public override Task<ListOfArticles> List(Empty request, ServerCallContext context)
    {
        var articles = _articleService.Articles;
        var listOfArticles = new ListOfArticles {Articles = { articles }};
        return Task.FromResult(listOfArticles);
    }
}
```

Next, register `BlogService` in the service collection.

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddGrpc();
        services.AddSingleton<ArticleService>();
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseRouting();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapGrpcService<BlogService>();
        });
    }
}
```

Also, I modified my `Program.cs`, because gRPC template uses TLS by default, but I don’t want it in my development environment. Here is the [article about configuring HTTP in ASP.NET Core across different platforms](https://devblogs.microsoft.com/aspnet/configuring-https-in-asp-net-core-across-different-platforms/).

```c#
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureKestrel(options =>
                {
                    options.ListenLocalhost(5000, o => o.Protocols = HttpProtocols.Http2);
                });
                webBuilder.UseStartup<Startup>();
            });
}
```

That’s all we need, let’s test our server. There is a handy tool for doing that called [BloomRPC](https://github.com/uw-labs/bloomrpc). If you are familiar to Postman, this tool is similar to it in the gRPC world.

![Create item with BloomRPC](/images/2020-01-29-grpc-with-asp-net-core/bloom-rpc.png)

![Get items with BloomRPC](/images/2020-01-29-grpc-with-asp-net-core/bloom-rpc-get.png)

# Client
It’s time to build a client. I used the same worker template as in the [previous post]({% post_url 2019-11-25-rest-api-with-asp-net-core %}). To continue with, I added some nuget libraries and proto file to the client project.

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
    <UserSecretsId>dotnet-Client-B21D299E-C071-4523-A770-9ECD8AE58793</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Google.Protobuf" Version="3.11.2" />
    <PackageReference Include="Grpc.Net.Client" Version="2.26.0" />
    <PackageReference Include="Grpc.Net.ClientFactory" Version="2.26.0" />
    <PackageReference Include="Grpc.Tools" Version="2.26.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="3.1.1" />
  </ItemGroup>

  <ItemGroup>
    <Protobuf Include="..\Blog.proto" GrpcServices="BlogClient" />
  </ItemGroup>

</Project>
```

After that, I registered `BlogClient` and added this line `AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true);` to call insecure gRPC services.

```c#
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureServices((hostContext, services) =>
            {
                AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true);
                services.AddHostedService<Worker>();
                services.AddGrpcClient<Blog.BlogClient>(o =>
                {
                    o.Address = new Uri("http://localhost:5000");
                });
            });
}
```

And there is our worker.

```c#
public class Worker : BackgroundService
{
    private const string CreateCommand = "create";
    private const string GetListCommand = "get-list";
    private readonly Blog.BlogClient _client;
    
    public Worker(Blog.BlogClient client)
    {
        _client = client;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("Type a command.");
            var command = Console.ReadLine();
            if (command != CreateCommand && command != GetListCommand)
            {
                Console.WriteLine("Wrong command. Try again.");
                continue;
            }
            
            if (command is GetListCommand)
            {
                var articles = await _client.ListAsync(new Empty());
                Console.WriteLine($"We have {articles.Articles.Count} articles");
                foreach (var article in articles.Articles)
                {
                    Console.WriteLine($"Article '{article.Name}' by {article.Author}");
                }
            }
            else if (command is CreateCommand)
            {
                Console.WriteLine("Type a name");
                var name = Console.ReadLine();
                Console.WriteLine("Type an author");
                var author = Console.ReadLine();
                Console.WriteLine("Type a content");
                var content = Console.ReadLine();

                var article = new Article {Name = name, Author = author, Content = content};
                _ = await _client.CreateAsync(article, cancellationToken: stoppingToken);
            }
        }
    }
}
```

Let’s make some calls.

![Console](/images/2020-01-29-grpc-with-asp-net-core/console.png)

Our server and client are done. As you can see, the creation of grpc communication is relatively simple. And this is great because you don’t need to make great efforts to learn a new approach. Everything will seem very similar.

# Streaming
The more advanced scenario is streaming. gRPC allows you to perform server-to-client, client-to-server and bidirectional streaming. For example, a client sends a request and receive a number of messages. It may be more efficient in the case of transferring a large amount of data or server’s pushes.

I show you modifications that need to be done to implement gRPC streaming. Firstly, add `stream` to your contract instead of repeated.

```protobuf
syntax = "proto3";

import "google/protobuf/empty.proto";

service BlogServer {
  rpc Create (Article) returns (google.protobuf.Empty);
  rpc List (google.protobuf.Empty) returns (stream Article);
}

message Article {
  string name = 1;
  string author = 2;
  string content = 3;
}
```

Secondly, modify `BlogService` and put a delay to simulate some work.

```c#
public class BlogService : BlogServer.BlogServerBase
{
    private readonly ArticleService _articleService;

    public BlogService(ArticleService articleService)
    {
        _articleService = articleService;
    }
    
    public override Task<Empty> Create(Article article, ServerCallContext context)
    {
        _articleService.Articles.Add(article);
        return Task.FromResult(new Empty());
    }

    public override async Task List(Empty request, IServerStreamWriter<Article> responseStream, ServerCallContext context)
    {
        var articles = _articleService.Articles;
        foreach (var article in articles)
        {
            if (context.CancellationToken.IsCancellationRequested)
            {
                break;
            }

            await responseStream.WriteAsync(article);

            await Task.Delay(1000);
        }
    }
}
```

Finally, change the client. I use new async streams to receive articles from the server.

```c#
public class Worker : BackgroundService
{
    private const string CreateCommand = "create";
    private const string GetListCommand = "get-list";
    private readonly BlogServer.BlogServerClient _client;
    
    public Worker(BlogServer.BlogServerClient client)
    {
        _client = client;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("Type a command.");
            var command = Console.ReadLine();
            if (command != CreateCommand && command != GetListCommand)
            {
                Console.WriteLine("Wrong command. Try again.");
                continue;
            }
            
            if (command is GetListCommand)
            {
                using var articles = _client.List(new Empty());
                await foreach (var article in articles.ResponseStream.ReadAllAsync(stoppingToken))
                {
                    Console.WriteLine($"Time: {DateTime.Now:HH:mm:ss}");
                    Console.WriteLine($"Article '{article.Name}' by {article.Author}");
                }
            }
            else if (command is CreateCommand)
            {
                Console.WriteLine("Type a name");
                var name = Console.ReadLine();
                Console.WriteLine("Type an author");
                var author = Console.ReadLine();
                Console.WriteLine("Type a content");
                var content = Console.ReadLine();

                var article = new Article {Name = name, Author = author, Content = content};
                await _client.CreateAsync(article, cancellationToken: stoppingToken);
            }
        }
    }
}
```

With the following screenshot, you can see that messages arrive at the client with a delay.

![Console with results](/images/2020-01-29-grpc-with-asp-net-core/console-results.png)

# Summary
In this post, I showed how to implement communication through gRPC in ASP.NET Core application. gRPC was designed for services calling, and it does its work pretty well. There are some fascinating features like streaming, backward compatibility, deadlines, error status codes, and they help you to be more productive. I didn’t tell you about most of them, because it is a basic post and they are well described in the [documentation](https://grpc.io/docs/guides/concepts/). There are also some drawbacks. For example, gRPS isn’t fully supported by browsers, and all messages are binary, so you need a tool to understand them while debugging.

You may think that gRPC is more efficient than REST, but it is not so simple. There are different benchmarks over the internet and in some scenarios REST is better. Eventually, if you need such a performance, maybe, you should refuse remote calls at all.

To summarize, gRPC is not a silver bullet, but it’s interesting and perspective technology in its area. You should give it a try if you’re working with microservices.

# References
* [Introduction to gRPC on .NET Core](https://docs.microsoft.com/en-us/aspnet/core/grpc/?view=aspnetcore-3.1);
* [gRPC vs HTTP APIs](https://devblogs.microsoft.com/aspnet/grpc-vs-http-apis/);
* [Steve Gordon blog](https://www.stevejgordon.co.uk/category/grpc);
* [Awesome gRPC](https://github.com/grpc-ecosystem/awesome-grpc);
* [Building Microservices with gRPC and .NET](https://www.youtube.com/watch?v=4anDV0zBIZw).

*Image: Photo by David Hellmann on Unsplash*



