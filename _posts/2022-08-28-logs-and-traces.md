---
title: "What does the .NET application say - Logs and Traces"
excerpt: "In this post, we’ll talk about hot to log and receive some information with the help of DiagnosticSource and EventSource."
header:
  overlay_image: /images/2022-08-28-logs-and-traces/cover.jpg
  show_overlay_excerpt: false
  caption: "Photo by [Scott Walsh](https://unsplash.com/@outsighted) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2022-08-28
tags: ["Logs", "Traces", "DiagnosticSource", "EventSource", ".NET"]
---
I think that the [logging](https://docs.microsoft.com/en-us/dotnet/core/extensions/logging) isn’t a new concept for .NET users. Many of us use `ILogger` interface in the application to save some diagnostic information. I’ve solved so many bugs with the help of logs. But every so often you can’t or don’t want to use `ILogger`, or you may need to obtain logs from .NET itself. In this post, we’ll discuss `DiagnosticSource` and `EventSource`, which information you can receive from them, how to use these classes for your scenarios.

This is the second part. In the [previous post]({% post_url 2022-06-02-counters-and-metrics %}), we talked about metrics and counters.

# `DiagnosticSource`

Let’s start with a [`DiagnosticSource`](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/diagnosticsource-diagnosticlistener). The main purpose of using it is an in-process logging. This class allows you to log complex non-serializable data structures to analyze them in the other parts of your program.

I’m going to create a simple example and then show you, how .NET use such type of logging.

## Producing

Firstly, create a new console application.

```
$ dotnet new console -n diagnostic-source
```

Thereafter, add an event that you will log.

```csharp
public record MyEvent(int Value)
{
    public const string Name = "MyEvent";
}
```

Finally, use a `DiagnosticSource` to produce these events.

```csharp
Console.WriteLine("DiagnosticSource sample application");

const string myDiagnosticSourceName = "Example.MyDiagnosticSource";
DiagnosticSource diagnosticSource = new DiagnosticListener(myDiagnosticSourceName);

Task.Run(async () => await StartEventProducingTask());

Console.ReadKey();

async Task StartEventProducingTask()
{
    var random = new Random();
    while (true)
    {
        if (diagnosticSource.IsEnabled(MyEvent.Name))
        {
            diagnosticSource.Write(MyEvent.Name, new MyEvent(random.Next(0, 10)));
        }

        await Task.Delay(200);
    }
}
```

Pretty straightforward code.

## In-process consuming

The next step is to catch our events. To achieve that, we need to add an `IObserver` which will filter other logs and somehow handle ours.

```csharp
public sealed class MyDiagnosticSourceObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(KeyValuePair<string, object?> value)
    {
        if (value.Key == MyEvent.Name)
        {
            var myEvent = (MyEvent?)value.Value;
            if (myEvent != null)
            {
                Console.WriteLine($"Event received: {myEvent.Value}");
            }
        }
    }
}
```

After that, we have to add another `IObserver` that will create a subscription for the `MyDiagnosticSourceObserver`.

```csharp
public sealed class AllDiagnosticListenerObserver : IObserver<DiagnosticListener>
{
    private const string MyDiagnosticSourceName = "Example.MyDiagnosticSource";
    private IDisposable? _myDiagnosticSourceSubscription;

    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(DiagnosticListener listener)
    {
        if (listener.Name == MyDiagnosticSourceName)
        {
            _myDiagnosticSourceSubscription ??= listener.Subscribe(new MyDiagnosticSourceObserver());
        }
    }
}
```

Lastly, subscribe `AllDiagnosticListenerObserver` to `AllListeners` property of `DiagnosticListener` class.

```csharp
using var subscription = DiagnosticListener.AllListeners.Subscribe(new AllDiagnosticListenerObserver());
```

After all, you’ll see that the random numbers from the `MyEvent` will appear on the console.

## Out-of-process consuming

Next, I’ll show how to receive `DiagnosticSource` logs from another process. Yes, this type of logging is mostly about in-process consuming. But in some cases you need to monitor them separately. I’ve described one such scenario in [my previous post]({% post_url 2021-09-22-tracing-for-messaging-application %}).

We’ll use two libraries, `Microsoft.Diagnostics.NETCore.Client` and `Microsoft.Diagnostics.Tracing.TraceEvent`. The solution will be very similar to what we did to consume metrics and counters.

```csharp
Console.WriteLine("DiagnosticSource listener sample application");
Console.WriteLine("Specify a target pid");
var input = Console.ReadLine();
if (!int.TryParse(input, out var pid))
{
    return;
}

var client = new DiagnosticsClient(pid);
var arguments = new Dictionary<string, string?> { ["FilterAndPayloadSpecs"] = "Example.MyDiagnosticSource/MyEvent:-Value" };
var providers = new List<EventPipeProvider>
{
    new("Microsoft-Diagnostics-DiagnosticSource", EventLevel.Verbose, (long)EventKeywords.All, arguments)
};
using var session = client.StartEventPipeSession(providers, false);
using var source = new EventPipeEventSource(session.EventStream);

Console.CancelKeyPress += (_, eventArgs) =>
{
    eventArgs.Cancel = true;
    session.Stop();
};

source.Dynamic.All += HandleEvent;
source.Process();

static void HandleEvent(TraceEvent evt)
{
    if (evt.EventName != "Event")
    {
        return;
    }

    var sourceName = evt.PayloadValue(0) as string;
    var eventName = evt.PayloadValue(1) as string;
    if (sourceName != "Example.MyDiagnosticSource" || eventName != "MyEvent")
    {
        return;
    }
    
    if (evt.PayloadValue(2) is not IDictionary<string, object>[] payload)
    {
        return;
    }

    var eventValue = payload[0]["Value"];

    Console.WriteLine($"Event received: {eventValue}");
}
```

The main trick here is that we’re using a special bridge [`Microsoft-Diagnostics-DiagnosticSource`](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/System/Diagnostics/DiagnosticSourceEventSource.cs) which proxies our events outside the main process. Here you can see how to specify providers and filters to get the data we needed.

```csharp
var arguments = new Dictionary<string, string?> { ["FilterAndPayloadSpecs"] = "Example.MyDiagnosticSource/MyEvent:-Value" };
var providers = new List<EventPipeProvider>
{
    new("Microsoft-Diagnostics-DiagnosticSource", EventLevel.Verbose, (long)EventKeywords.All, arguments)
};
```

It isn’t a perfect solution, you won’t receive complex types, only the ones that can be serialized. But even that can help you when you try to analyze some bugs.

## Another `DiagnosticSources`

Now, let’s try to find another available `DiagnosticSource` in our application. Create a new `webapi` application and modify our `AllDiagnosticListenerObserver`.

```csharp
public sealed class AllDiagnosticListenerObserver : IObserver<DiagnosticListener>
{
    private readonly ConcurrentDictionary<string, IDisposable> _subscription = new();

    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(DiagnosticListener listener)
    {
        Console.WriteLine($"Listener found: {listener.Name}");
        _subscription.TryAdd(listener.Name, listener.Subscribe(new CommonDiagnosticSourceObserver()));
    }
}
```

The `CommonDiagnosticSourceObserver` will be like this:

```csharp
public sealed class CommonDiagnosticSourceObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(KeyValuePair<string, object?> value)
    {
        Console.WriteLine($"Event received: {value.Key}, {value.Value?.GetType()}");
    }
}
```

Start the application, and you’ll see such sources and events.

```
Listener found: Microsoft.Extensions.Hosting
Event received: HostBuilding, Microsoft.Extensions.Hosting.HostBuilder
Event received: HostBuilt, Microsoft.Extensions.Hosting.Internal.Host
Listener found: Microsoft.AspNetCore
```

Send a request `GET http://localhost:5000/WeatherForecast`, and you’ll see more events.

```
Event received: Microsoft.AspNetCore.Hosting.HttpRequestIn.Start, Microsoft.AspNetCore.Http.DefaultHttpContext
Event received: Microsoft.AspNetCore.Hosting.BeginRequest, <>f__AnonymousType0`2[Microsoft.AspNetCore.Http.HttpContext,System.Int64]
Event received: Microsoft.AspNetCore.Routing.EndpointMatched, Microsoft.AspNetCore.Http.DefaultHttpContext
Event received: Microsoft.AspNetCore.Mvc.BeforeAction, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeOnActionExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionFilterOnActionExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnActionExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionFilterOnActionExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeOnActionExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionFilterOnActionExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnActionExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionFilterOnActionExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeActionMethod, <>f__AnonymousType0`3[Microsoft.AspNetCore.Mvc.ActionContext,System.Collections.Generic.IReadOnlyDictionary`2[System.String,System.Object],System.Object]
Event received: Microsoft.AspNetCore.Mvc.BeforeControllerActionMethod, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeControllerActionMethodEventData
Event received: Microsoft.AspNetCore.Mvc.AfterControllerActionMethod, Microsoft.AspNetCore.Mvc.Diagnostics.AfterControllerActionMethodEventData
Event received: Microsoft.AspNetCore.Mvc.AfterActionMethod, <>f__AnonymousType1`4[Microsoft.AspNetCore.Mvc.ActionContext,System.Collections.Generic.IReadOnlyDictionary`2[System.String,System.Object],System.Object,Microsoft.AspNetCore.Mvc.IActionResult]
Event received: Microsoft.AspNetCore.Mvc.BeforeOnActionExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionFilterOnActionExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnActionExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionFilterOnActionExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeOnActionExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionFilterOnActionExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnActionExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionFilterOnActionExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeOnResultExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeResultFilterOnResultExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnResultExecuting, Microsoft.AspNetCore.Mvc.Diagnostics.AfterResultFilterOnResultExecutingEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeActionResult, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeActionResultEventData
Event received: Microsoft.AspNetCore.Mvc.AfterActionResult, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionResultEventData
Event received: Microsoft.AspNetCore.Mvc.BeforeOnResultExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.BeforeResultFilterOnResultExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.AfterOnResultExecuted, Microsoft.AspNetCore.Mvc.Diagnostics.AfterResultFilterOnResultExecutedEventData
Event received: Microsoft.AspNetCore.Mvc.AfterAction, Microsoft.AspNetCore.Mvc.Diagnostics.AfterActionEventData
Event received: Microsoft.AspNetCore.Hosting.EndRequest, <>f__AnonymousType0`2[Microsoft.AspNetCore.Http.HttpContext,System.Int64]
Event received: Microsoft.AspNetCore.Hosting.HttpRequestIn.Stop, Microsoft.AspNetCore.Http.DefaultHttpContext
```

Lots of events and valuable diagnostics information that you can use for your purposes.

Now, imagine, that we need to understand the current `EnvironmentName` (or even change it). To achieve this, create a new `IObserver` and add a subscription for it in the `AllDiagnosticListenerObserver`.

```csharp
if (listener.Name == "Microsoft.Extensions.Hosting")
{
    _subscription.TryAdd("CustomHostingObserver", listener.Subscribe(new MicrosoftHostingObserver()));
}
```

```csharp
public sealed class MicrosoftHostingObserver : IObserver<KeyValuePair<string, object?>>
{
    public void OnCompleted()
    {
    }

    public void OnError(Exception error)
    {
    }

    public void OnNext(KeyValuePair<string, object?> value)
    {
        if (value.Key == "HostBuilding")
        {
            var hostBuilder = (HostBuilder?)value.Value;
            var context = hostBuilder?.Properties[typeof(WebHostBuilderContext)];
            if (context is not WebHostBuilderContext webHostBuilderContext)
            {
                return;
            }
            
            Console.WriteLine($"My hosting environment: {webHostBuilderContext.HostingEnvironment.EnvironmentName}");
        }
    }
}
```

Further, in [this great post](https://andrewlock.net/exploring-dotnet-6-part-5-supporting-ef-core-tools-with-webapplicationbuilder/), you can find another example how EF Core uses `DiagnosticSource` for their needs.

# `EventSource`

Continuing on, we move further and talk about `EventSource`.

> `System.Diagnostics.Tracing.EventSource` is a fast structured logging solution built into the .NET runtime.

It’s widely used in different parts of the runtime. So let's look at an example.

## Producing

Foremost, we require a new `EventSource`.

```csharp
[EventSource(Name = "Example.MyEventSource")]
public sealed class MyEventSource : EventSource
{
    public static readonly MyEventSource Instance = new();

    [Event(1)]
    public void ValueReceived(int value) => WriteEvent(1, value);
}
```

Then, logging will be simple:

```csharp
Console.WriteLine("EventSource sample application");

Task.Run(async () => await StartEventProducingTask());

Console.ReadKey();

async Task StartEventProducingTask()
{
    var random = new Random();
    while (true)
    {
        MyEventSource.Instance.ValueReceived(random.Next(0, 10));
        await Task.Delay(200);
    }
}
```

## In-process consuming

To consume events, we have to create an inheritor of `EventListener` class. And everything is ready.

```csharp
public sealed class MyEventListener : EventListener
{
    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name == "Example.MyEventSource")
        {
            EnableEvents(eventSource, EventLevel.Informational);
        }
    }

    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        if (eventData.EventName != nameof(MyEventSource.ValueReceived) || eventData.Payload is null)
        {
            return;
        }

        Console.WriteLine($"Event received: {eventData.Payload[0]}");
    }
}
```

## Out-of-process consuming

Here is the snippet to consume such events from another process.

```csharp
Console.WriteLine("EventSource listener sample application");
Console.WriteLine("Specify a target pid");
var input = Console.ReadLine();
if (!int.TryParse(input, out var pid))
{
    return;
}

var client = new DiagnosticsClient(pid);
var providers = new List<EventPipeProvider>
{
    new("Example.MyEventSource", EventLevel.Verbose, (long)EventKeywords.All)
};
using var session = client.StartEventPipeSession(providers, false);
using var source = new EventPipeEventSource(session.EventStream);

Console.CancelKeyPress += (_, eventArgs) =>
{
    eventArgs.Cancel = true;
    session.Stop();
};

source.Dynamic.All += HandleEvent;
source.Process();

static void HandleEvent(TraceEvent evt)
{
    if (evt.EventName != "ValueReceived")
    {
        return;
    }

    var eventValue = evt.PayloadValue(0);

    Console.WriteLine($"Event received: {eventValue}");
}
```

As you see, this is a very similar example to what we have done in other out-of-process cases. But now I’ve changed a provider:

```csharp
var providers = new List<EventPipeProvider>
{
    new("Example.MyEventSource", EventLevel.Verbose, (long)EventKeywords.All)
};
```

### Tools

Another way to consume logs is using tools: [PerfView](https://github.com/microsoft/perfview), [dotnet-trace](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) or [my plugin](https://plugins.jetbrains.com/plugin/19141-diagnostics-client) for JetBrains Rider. This method allows you, for example, to save events to a file and analyze them later. A very handy feature, especially for production. I won’t touch this topic in the current post, but you can find some examples in the [documentation](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/eventsource-collect-and-view-traces#perfview).

## Another `EventSources`

Where to find available `EventSources`? Here are some useful links: [Well-known event providers in .NET](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/well-known-event-providers), [.NET runtime events](https://docs.microsoft.com/en-us/dotnet/fundamentals/diagnostics/runtime-events), [CLR ETW Events](https://docs.microsoft.com/en-us/dotnet/framework/performance/clr-etw-events).

Below, I want to give some cases on how to collect from different `EventSources`.

1. Collect logs from `ILogger`:

```
$ dotnet trace collect -p {PID} --providers Microsoft-Extensions-Logging:4:5
```

2. Collect events about exception:

```
$ dotnet trace collect -p {PID} --providers Microsoft-Windows-DotNETRuntime:0x8000:4
```

3. Collect events from Kestrel:

```
$ dotnet trace collect -p {PID} --providers Microsoft-AspNetCore-Server-Kestrel::4
```

4. Collect events about GC:

```
$ dotnet trace collect -p {PID} --providers Microsoft-Windows-DotNETRuntime:0x1:4
```

5. Collect events from the thread pool:

```
$ dotnet trace collect -p {PID} --providers Microsoft-Windows-DotNETRuntime:0x10000:4
```

If you think I've missed any popular providers, please, write them in the comments.

# Conclusion

It's a pretty long post. Despite this, I did not talk about several things, such as [EvenPipes](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/eventpipe) and tooling. Maybe I'll come back to this in future posts 🙂.

Nonetheless, I’ve shown some code examples how to produce and consume logs via `DiagnosticSource` and `EventSource`. Perhaps this knowledge is not useful in day-to-day work, but it may help you to analyze non-trivial bugs or even build a custom diagnostics system to control your applications.

# References

*The source code is available [here](https://github.com/rafaelldi/dotnet-diagnostics-events)*

- [What does the .NET application say - Counters and Metrics](% post_url 2022-06-02-counters-and-metrics %)
- [.NET logging and tracing](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/logging-tracing)
- [Microsoft-Diagnostics-DiagnosticSource](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/System/Diagnostics/DiagnosticSourceEventSource.cs)
- [Well-known event providers in .NET](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/well-known-event-providers)
- [Logging in .NET Core and ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-6.0#event-source)
- [.NET runtime events](https://docs.microsoft.com/en-us/dotnet/fundamentals/diagnostics/runtime-events)
- [CLR ETW Events](https://docs.microsoft.com/en-us/dotnet/framework/performance/clr-etw-events)