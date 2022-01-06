---
title: "Monitoring background task"
excerpt: "I'm continuing my previous post about asynchronous processing. I want to show how to add some metrics to monitor background tasks."
categories: posts
author: Rival Abdrakhmanov
date: 2021-07-21
tags: ["Distributed application", "MassTransit", "Messaging", "ASP.NET Core", "EventCounters", "Diagnostics", "Performance"]
---

![Title image](/images/2021-07-21-monitoring-background-task/cover_monitoring_task.jpg)

When you have async processing, it may be good to know how this task is performing. Is this process quick enough, or how many times was it executed? In this post, I'm going to add simple metrics with [EventCounters](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/event-counters).

# EventCounters
Since .NET Core 3.0, we have a simple, lightweight mechanism to monitor applications. You don't need to install any library; everything is available automatically. To take a look at these metrics, you can use the [dotnet-counters tool](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters). Firstly, install it via console.

```
$ dotnet tool install --global dotnet-counters
```

Secondly, choose the available dotnet process.

```
$ dotnet-counters ps
```

Finally, monitor its metrics.

```
$ dotnet-counters monitor --process-id 132378
```

In the output, you'll see something like this.

```
[System.Runtime]
    % Time in GC since last GC (%)                                 0    
    Allocation Rate (B / 1 sec)                               65,320    
    CPU Usage (%)                                                  0    
    Exception Count (Count / 1 sec)                                0    
    GC Fragmentation (%)                                         NaN    
    GC Heap Size (MB)                                              9    
    Gen 0 GC Count (Count / 1 sec)                                 0    
    Gen 0 Size (B)                                                 0    
    Gen 1 GC Count (Count / 1 sec)                                 0    
    Gen 1 Size (B)                                                 0    
    Gen 2 GC Count (Count / 1 sec)                                 0    
    Gen 2 Size (B)                                                 0    
    IL Bytes Jitted (B)                                      269,318    
    LOH Size (B)                                                   0    
    Monitor Lock Contention Count (Count / 1 sec)                  0    
    Number of Active Timers                                        2    
    Number of Assemblies Loaded                                  131    
    Number of Methods Jitted                                   4,483    
    POH (Pinned Object Heap) Size (B)                              0    
    ThreadPool Completed Work Item Count (Count / 1 sec)           0    
    ThreadPool Queue Length                                        0    
    ThreadPool Thread Count                                        4    
    Working Set (MB)                                             101
```

# Implementing your own metric

The good part is that we can quickly introduce our own metric. Let's imagine that we need to know the average duration of the step in our process.

There are two main types of metrics: `EventCounter` and `IncrementingEventCounter`. The first one is to calculate a statistical summary for a set of values in the interval. The second type of metric is to calculate the total in the interval. So, the `EventCounter` type is appropriate for our scenario.

```c#
[EventSource(Name = "AsyncRequestProcessing.ActivityEventCounters")]
public sealed class ActivityEventCountersSource : EventSource
{
    public static readonly ActivityEventCountersSource Instance = new();
    private EventCounter _processingTimeCounter;

    private ActivityEventCountersSource()
    {
        _processingTimeCounter = new EventCounter("processing-time", this)
        {
            DisplayName = "Item Processing Time",
            DisplayUnits = "ms"
        };
    }

    public void Log(long elapsedMilliseconds)
    {
        _processingTimeCounter?.WriteMetric(elapsedMilliseconds);
    }

    protected override void Dispose(bool disposing)
    {
        _processingTimeCounter?.Dispose();
        _processingTimeCounter = null;

        base.Dispose(disposing);
    }
}
```

To implement our metric, we need to create a new `EventSource` type. With `Log(long elapsedMilliseconds)` method, we'll write down each measurement. The next question is how to calculate the duration of the execution. MassTransit has [middlewares](https://masstransit-project.com/advanced/middleware/), like ASP.NET Core, so it's a perfect candidate for our purpose. Let's implement a straightforward filter.

```c#
public class EventCountersFilter<T> : IFilter<ExecuteContext<T>> where T : class
{
    private readonly Stopwatch _stopwatch = new();
    public async Task Send(ExecuteContext<T> context, IPipe<ExecuteContext<T>> next)
    {
        _stopwatch.Start();
        await next.Send(context);
        _stopwatch.Stop();

        ActivityEventCountersSource.Instance.Log(_stopwatch.ElapsedMilliseconds);
    }

    public void Probe(ProbeContext context)
    {
        context.CreateFilterScope("event-counters");
    }
}
```

```c#
services.AddMassTransit(x =>
    {
        ...

        x.UsingRabbitMq((context, cfg) =>
        {
            cfg.UseExecuteActivityFilter(typeof(EventCountersFilter<>), context);

            cfg.ConfigureEndpoints(context);
        });
    })
```

Now, everything is ready to check our new metric. Run `dotnet-counters` tool and specify `--counters` argument.

```
$ dotnet-counters monitor --process-id 65267 --counters AsyncRequestProcessing.ActivityEventCounters
```

After that, you'll see this report.

```
[AsyncRequestProcessing.ActivityEventCounters]
    Item Processing Time (ms)                      4,008.5
```

You can find the project in [my repository](https://github.com/rafaelldi/async-request-processing).

# Summary

In this post, I've shown how to add simple metrics to your application with little effort. It may be very convenient for testing performance in the development environment.

# References

- [EventCounters in .NET Core](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/event-counters)
- [Tutorial: Measure performance using EventCounters in .NET Core](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/event-counter-perf)
- [MassTransit Middleware](https://masstransit-project.com/advanced/middleware/)

*Image: Photo by Kent Pilcher on Unsplash*