---
layout: post
title: "ASP.NET Core: Logs (Part 1)"
categories: post
author: Rival Abdrakhmanov
date: 2019-07-14
tags: ["Observability", "ASP.NET Core", "Logging", "Serilog", "Structured Logging"]
---
Logging is one of the most important things in your application because it helps you to understand what happens in production, to find the roots of exceptions and to trace some behaviors. Today we’ll take a look at logging in ASP.NET Core. 

![Title image](/images/2019-07-14-asp-net-core-logs-part-1/cover_asp_net_core_logs_part_1.jpg)

# Structured logging
You may think that logging is highly inconvenient: your application writes something in a file, then you have to find this information in a tremendous volume of text. It doesn’t sound cool. But we can add some useful structures instead of logging only message. The following examples show the difference.

```
info: Microsoft.AspNetCore.Hosting.Internal.WebHost[1]
      Request starting HTTP/1.1 GET https://localhost:5001/api/values/1  
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[0]
      Executing endpoint 'WebApplication1.Controllers.ValuesController.Get (WebApplication1)'
info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[1]
      Route matched with {action = "Get", controller = "Values"}. Executing action WebApplication1.Controllers.ValuesController.Get (WebApplication1)
info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[1]
      Executing action method WebApplication1.Controllers.ValuesController.Get (WebApplication1) with arguments (1) - Validation state: Valid
info: WebApplication1.Controllers.ValuesController[0]
      Id value is 1
info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[2]
      Executed action method WebApplication1.Controllers.ValuesController.Get (WebApplication1), returned result Microsoft.AspNetCore.Mvc.ObjectResult in 5.1358ms.
info: Microsoft.AspNetCore.Mvc.Infrastructure.ObjectResultExecutor[1]
      Executing ObjectResult, writing value of type 'System.String'.
info: Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker[2]
      Executed action WebApplication1.Controllers.ValuesController.Get (WebApplication1) in 295.2989ms
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[1]
      Executed endpoint 'WebApplication1.Controllers.ValuesController.Get (WebApplication1)'
info: Microsoft.AspNetCore.Hosting.Internal.WebHost[2]
      Request finished in 657.7785ms 200 application/json; charset=utf-8
```

```json lines
{"@t":"2019-07-13T14:28:14.0957104Z","@mt":"{HostingRequestStartingLog:l}","@r":["Request starting HTTP/1.1 GET http://localhost:5000/api/values/1  "],"Protocol":"HTTP/1.1","Method":"GET","ContentType":null,"ContentLength":null,"Scheme":"http","Host":"localhost:5000","PathBase":"","Path":"/api/values/1","QueryString":"","HostingRequestStartingLog":"Request starting HTTP/1.1 GET http://localhost:5000/api/values/1  ","EventId":{"Id":1},"SourceContext":"Microsoft.AspNetCore.Hosting.Internal.WebHost","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.1233725Z","@mt":"Executing endpoint '{EndpointName}'","EndpointName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","EventId":{"Name":"ExecutingEndpoint"},"SourceContext":"Microsoft.AspNetCore.Routing.EndpointMiddleware","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.1625562Z","@mt":"Route matched with {RouteData}. Executing controller action with signature {MethodInfo} on controller {Controller} ({AssemblyName}).","RouteData":"{action = \"Get\", controller = \"Values\"}","MethodInfo":"Microsoft.AspNetCore.Mvc.ActionResult`1[System.String] Get(Int32)","Controller":"AspNetCoreAppLogging.Controllers.ValuesController","AssemblyName":"AspNetCoreAppLogging","EventId":{"Id":3},"SourceContext":"Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.2040323Z","@mt":"Executing action method {ActionName} - Validation state: {ValidationState}","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","ValidationState":"Valid","EventId":{"Id":1},"SourceContext":"Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.2052423Z","@mt":"Id value is {@id}","id":1,"SourceContext":"AspNetCoreAppLogging.Controllers.ValuesController","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.2082649Z","@mt":"Executed action method {ActionName}, returned result {ActionResult} in {ElapsedMilliseconds}ms.","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","ActionResult":"Microsoft.AspNetCore.Mvc.ObjectResult","ElapsedMilliseconds":1.2669000000000001,"EventId":{"Id":2},"SourceContext":"Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.2165533Z","@mt":"Executing ObjectResult, writing value of type '{Type}'.","Type":"System.String","EventId":{"Id":1},"SourceContext":"Microsoft.AspNetCore.Mvc.Infrastructure.ObjectResultExecutor","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.3131992Z","@mt":"Executed action {ActionName} in {ElapsedMilliseconds}ms","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","ElapsedMilliseconds":148.5523,"EventId":{"Id":2},"SourceContext":"Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker","ActionId":"6855c1f5-6601-4386-94ed-2c056336e313","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.3135792Z","@mt":"Executed endpoint '{EndpointName}'","EndpointName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","EventId":{"Id":1,"Name":"ExecutedEndpoint"},"SourceContext":"Microsoft.AspNetCore.Routing.EndpointMiddleware","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
{"@t":"2019-07-13T14:28:14.3180543Z","@mt":"{HostingRequestFinishedLog:l}","@r":["Request finished in 223.0883ms 200 application/json; charset=utf-8"],"ElapsedMilliseconds":223.0883,"StatusCode":200,"ContentType":"application/json; charset=utf-8","HostingRequestFinishedLog":"Request finished in 223.0883ms 200 application/json; charset=utf-8","EventId":{"Id":2},"SourceContext":"Microsoft.AspNetCore.Hosting.Internal.WebHost","RequestId":"0HLO7JPBE4OS4:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO7JPBE4OS4"}
```

The first example looks like a plain text, the second one has a json structure with different fields. Of course, if you’re looking in the debug console, the first log is more human-readable, but you can add the json-files into some storage and easily parse them. Later I’ll show you some details.

# Serilog
[Serilog](https://serilog.net/) is a package for structured logging in .NET. It has a lot of plugins for different customizations.

# Console logging
Firstly, let’s look at the simple scenario when you write logs to the console. To start with, you need to install the following packages:
1. [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore);
2. [Serilog.Sinks.Console](https://github.com/serilog/serilog-sinks-console);
3. [Serilog.Settings.Configuration](https://github.com/serilog/serilog-settings-configuration).

AspNetCore nuget enables you to replace default `ILogger` with a Serilog. Sinks set up destinations to your logs. We use a console, it also may be a file, elasticsearch, seq, slack and [many others](https://www.nuget.org/packages?q=Tags%3A%22serilog%22). With configuration nuget, you can provide settings for Serilog from different [Microsoft.Extensions.Configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2) sources, for example, `appsettings.json` file.

After that, write your config file where specify console destination.

```json
"Serilog":{
  "MinimumLevel": {
    "Default": "Information"
  },
  "WriteTo":[{
    "Name": "Console"
  }
  ]
}
```

Initialize logger in `Program.cs` file and set up it with your config.

```c#
public class Program
{
    public static void Main(string[] args)
    {
        var configuration = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json")
            .AddEnvironmentVariables()
            .Build();
        Log.Logger = new LoggerConfiguration()
            .ReadFrom.Configuration(configuration)
            .Enrich.FromLogContext()
            .CreateLogger();

        try
        {
            Log.Information("Starting");
            CreateWebHostBuilder(args).Build().Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Host terminated unexpectedly");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseSerilog()
            .UseStartup<Startup>();
}
```

Although it is a recommended approach, you can do it in a [shorter way](https://github.com/serilog/serilog-aspnetcore#inline-initialization).

```c#
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseSerilog((hostingContext, loggerConfiguration) => loggerConfiguration
                .ReadFrom.Configuration(hostingContext.Configuration))
            .UseStartup<Startup>();
}
```

> This has the advantage of making the hostingContext‘s Configuration object available for configuration of the logger, but at the expense of recording Exceptions raised earlier in program startup.

That’s all for the setting up, and now you can pass logger through standard DI and write your logs with it.

```c#
private readonly ILogger<ValuesController> _logger;

public ValuesController(ILogger<ValuesController> logger)
{
    _logger = logger;
}
```

```c#
public ActionResult<string> Get(int id)
{
    var currentDate = DateTime.Now;
    _logger.LogInformation("Id value is {@id}", id);
    _logger.LogInformation("CurrentDate {@currentDate}", currentDate);
    return "value";
}
```

```
[12:27:51 INF] Id value is 1
[12:27:51 INF] CurrentDate 07/14/2019 12:27:51
```

Take a look at the logging form: `“Id value is {@id}”`, id.  It allows Serilog to save different parameters along with the message. It isn’t visible in a simple console log, so we add a formatter. As I said, Serilog has many extensions. We’ll use [this formatter](https://github.com/serilog/serilog-formatting-compact) to create a json structure from our message. Install [Serilog.Formatting.Compact](https://github.com/serilog/serilog-formatting-compact) package and make some changes in the config.

```json
"Serilog":{
  "MinimumLevel": {
    "Default": "Information"
  },
  "WriteTo":[{
    "Name": "Console",
    "Args": {
      "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
    }
  }
  ]
}
```
```json lines
{"@t":"2019-07-14T09:44:50.5953439Z","@mt":"Id value is {@id}","id":1,"SourceContext":"AspNetCoreAppLogging.Controllers.ValuesController","ActionId":"06ed54d2-f0d4-4586-b545-52f3d7c2b95b","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","RequestId":"0HLO87VL5R9O1:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO87VL5R9O1"}
{"@t":"2019-07-14T09:44:50.5954389Z","@mt":"CurrentDate {@currentDate}","currentDate":"2019-07-14T12:44:50.5951098+03:00","SourceContext":"AspNetCoreAppLogging.Controllers.ValuesController","ActionId":"06ed54d2-f0d4-4586-b545-52f3d7c2b95b","ActionName":"AspNetCoreAppLogging.Controllers.ValuesController.Get (AspNetCoreAppLogging)","RequestId":"0HLO87VL5R9O1:00000001","RequestPath":"/api/values/1","CorrelationId":null,"ConnectionId":"0HLO87VL5R9O1"}
```

As you can see, we save message and parameters in different fields:

```
"@mt":"Id value is {@id}","id":1.
```

We’ll see all the advantages when we’ll use a more complicated system for storing our logs.

## Starting information
When your application is started, you see this information in the console without any formatting.

```
Hosting environment: Development
Content root path: /home/rafaelldi/RiderProjects/AspNetCoreAppLogging
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

We can deal with it by adding this line to our `Program.cs` file.

```c#
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .SuppressStatusMessages(true)
        .UseSerilog()
        .UseStartup<Startup>();
```

**Update**: In .NET Core 3.0 you no longer need to hide this information, because now it has structure. More details you can find in [this article](https://andrewlock.net/new-in-aspnetcore-3-structured-logging-for-startup-messages/) from Andrew Lock.

## Enrichers
Serilog allows you to add some additional information to your logs which may be very useful when you trying to understand what happened with your application. You can do it with special enrichers, let’s take a look at some of them.

1. [Serilog.Enrichers.Thread](https://github.com/serilog/serilog-enrichers-thread);
2. [Serilog.Enrichers.Process](https://github.com/serilog/serilog-enrichers-process);
3. [Serilog.Enrichers.Environment](https://github.com/serilog/serilog-enrichers-environment);
4. [Serilog.Exceptions](https://github.com/RehanSaeed/Serilog.Exceptions).

```c#
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information"
    },
    "WriteTo": [
      { "Name": "Console", "Args": { "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact" }}
    ],
    "Enrich": [ "WithThreadId", "WithThreadName", "WithProcessId", "WithProcessName", "WithMachineName", "WithEnvironmentUserName", "WithExceptionDetails" ]
  },
  "AllowedHosts": "*"
}
```

After that, Serilog will add to your logs some properties:

```
"ThreadId":10,"ProcessId":21338,"ProcessName":"dotnet","MachineName":"null"
```

More information you can find in their repositories.

# Summary
That’s it for today, we’ve discussed structured logging, created a basic application. More complicated examples from the future posts will be based on it. Before we move on you can see this simple app on [my Github](https://github.com/rafaelldi/AspNetCoreAppLogging) in a master branch.

*Image: Photo by Markus Spiske on Unsplash*