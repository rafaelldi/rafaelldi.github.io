---
title: "Trace your ASP.NET Core and EF Core application"
excerpt: "In this article, I want to show you how to use tracing in your applications and what benefits you can get from
it."
header:
  overlay_image: /assets/images/2023-06-19-trace-asp-net-and-ef-app/cover.jpg
  show_overlay_excerpt: false
  caption: "Photo by [Shaun Darwood](https://unsplash.com/@shaundarwood) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2023-06-19
tags: ["ASP.NET Core", "EF Core", "Traces", "Diagnostics", "DiagnosticListener"]
sidebar:
nav: "diagnostics"
---
This post is more of a practical example of my [previous one about traces]({% post_url 2022-08-28-logs-and-traces %}).
Let's look at where you can use them in your real-world projects and what superpowers they can give you.

# EF Core

I'll start with EF Core because it's easier to use tracing with it. Let's create a new application with EF Core just to
test our trace scenarios. It doesn't really matter what the code does. These tools can also be used for large-scale
projects.

As I showed in my [previous post]({% post_url 2022-08-28-logs-and-traces %}), to consume traces from the same process,
you need to create two observers.

```csharp
public class AllDiagnosticListenerObserver : IObserver<DiagnosticListener>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(DiagnosticListener listener)
    {
        if (listener.Name == DbLoggerCategory.Name /*"Microsoft.EntityFrameworkCore"*/)
        {
            listener.Subscribe(new EfCoreObserver());
        }
    }
}
```

Here you can see that EF Core has a great diagnostic API because you can use predefined constants (
e.g. `DbLoggerCategory.Name`), strongly typed objects and
the [documentation](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/diagnostic-listeners) can help
you a lot.

Then, subscribe this listener.

```csharp
DiagnosticListener.AllListeners.Subscribe(new AllDiagnosticListenerObserver());
```

Finally, let's add our custom observer and see what we can get from it.

```csharp
public class EfCoreObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(KeyValuePair<string, object?> pair)
    {
        //...
    }
}
```

You can then
use [`CoreEventId`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.coreeventid?view=efcore-7.0),
[`RelationalEventId`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.relationaleventid?view=efcore-7.0)
or some provider-specific custom events (
e.g. [`SqlServerEventId`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.sqlservereventid?view=efcore-7.0)).
Just check the documentation, it contains a good description of each event and the fields that can be observed in the
event.

For
example, [`RelationalEventId.CommandExecuted`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.relationaleventid.commandexecuted?view=efcore-7.0).
From this
event, [you can retrieve](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.commandexecutedeventdata?view=efcore-7.0)
the command that was executed, or the duration of the execution, or the result of the execution. Then analyse them in
some way and send alerts or something. Or you could connect to a running server and check the exact query it sends to
the database for a particular web request.

```csharp
if (pair.Key == RelationalEventId.CommandExecuted.Name)
{
    var payload = pair.Value as CommandExecutedEventData;
    var command = payload.Command.CommandText;
    var duration = payload.Duration;
    var result = payload.Result;
    //...
}
```

Or the next
example [`RelationalEventId.DataReaderClosing`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.relationaleventid.datareaderclosing?view=efcore-7.0).
[From it](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.datareaderclosingeventdata?view=efcore-7.0),
you can see the exact number of read operations that have been performed. Too many for one query? Let's log the command
itself!

```csharp
if (pair.Key == RelationalEventId.DataReaderClosing.Name)
{
    var payload = pair.Value as DataReaderClosingEventData;
    var readCount = payload.ReadCount;
    var command = payload.Command.CommandText;
    //...
}
```

Or [`CoreEventId.SaveChangesCompleted`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.coreeventid.savechangescompleted?view=efcore-7.0)
to trace the number of saved entities. You can use it to easily spot bugs when a command is executed with the wrong
`WHERE` clause and updates half of your database.
Or [`CoreEventId.SaveChangesFailed`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.coreeventid.savechangesfailed?view=efcore-7.0)
to check the cause of failure. Also, there are some warning you can monitor (
e.g [`CoreEventId.FirstWithoutOrderByAndFilterWarning`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.coreeventid.firstwithoutorderbyandfilterwarning?view=efcore-7.0)).
So you can see that tracing gives you access to extremely useful data that can help you create your own monitoring tools
and diagnose problems.

# ASP.NET Core

With ASP .NET Core this is not so easy as there are no predefined event names and payload types. How to find what is
available? First of all, you can put a breakpoint on `AllDiagnosticListenerObserver` and check which listeners are
active in your application. I found that there is one `Microsoft.AspNetCore`, let's subscribe to it.

```csharp
else if (listener.Name == "Microsoft.AspNetCore")
{
    listener.Subscribe(new AspNetCoreObserver());
}
```

```csharp
public class AspNetCoreObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(KeyValuePair<string, object?> pair)
    {
        var key = pair.Key;
    }
}
```

The next step is to similarly place a breakpoint on the new `AspNetCoreObserver` class and check event names and their
payloads.

![Debugging ASP.NET Core observer](/assets/images/2023-06-19-trace-asp-net-and-ef-app/debug-asp-net-observer.png)

For example, from `Microsoft.AspNetCore.Hosting.HttpRequestIn.Start`
and `Microsoft.AspNetCore.Hosting.HttpRequestIn.Stop` you can get `DefaultHttpContext` and all information about the
request.

```csharp
if (key == "Microsoft.AspNetCore.Hosting.HttpRequestIn.Start")
{
    var payload = pair.Value as DefaultHttpContext;
    var method = payload.Request.Method;
    var path = payload.Request.Path;
}
else if (key == "Microsoft.AspNetCore.Hosting.HttpRequestIn.Stop")
{
    var payload = pair.Value as DefaultHttpContext;
    var statusCode = payload.Response.StatusCode;
}
```

Or you can analyse exceptions. I use reflection here because this event has a payload as an anonymous type, and it isn’t
possible to just cast it.

```csharp
else if (key == "Microsoft.AspNetCore.Diagnostics.HandledException")
{
    var exceptionProperty = pair.Value.GetType().GetProperty("exception");
    var exception = exceptionProperty.GetValue(pair.Value) as Exception;
}
```

# Conclusion

In this post, I’ve shown some practical examples of using traces to monitor your application. You can use these
observers to constantly analyse events, or you can use a sidecar container to attach and capture events in a test or
staging environment. Or you could even have a console application to connect to your application only when you're
investigating some suspicious scenario to see what's going on inside. I think such a tool can save you a lot of time
when debugging a tricky problem.

# References

- [What does the .NET application say - Logs and Traces]({% post_url 2022-08-28-logs-and-traces %})
- [Using Diagnostic Listeners in EF Core](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/diagnostic-listeners)
- [CoreEventId Class](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.coreeventid?view=efcore-7.0)
- [RelationalEventId Class](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.relationaleventid?view=efcore-7.0)
- [DiagnosticSource User's Guide](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/DiagnosticSourceUsersGuide.md)