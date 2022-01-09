---
title: "Tracing for messaging application"
excerpt: "In the last post, we discussed a dotnet-counters tool. This one is continuing that diagnostics conversation with a dotnet-trace tool."
header:
  og_image: /images/2021-09-22-tracing-for-messaging-application/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2021-09-22
tags: ["Distributed application", "MassTransit", "Messaging", "ASP.NET Core", "Diagnostics", "Tracing"]
---

![Title image](/images/2021-09-22-tracing-for-messaging-application/cover.jpg)

Tracing can be very powerful to debug problems in production. I won't go deeper into the tracing details today, but I want to demonstrate one particular issue I encountered during development.

As I mentioned in the introduction, I'm going to discuss the [dotnet-trace](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) tool and [MassTransit](https://masstransit-project.com) library. The tool allows you to capture events from the application. This process is very similar to logging. Framework and many libraries write useful information with these events, and you can catch them and analyse the application's problems. MassTransit indeed writes some [diagnostic events](https://masstransit-project.com/advanced/monitoring/diagnostic-source.html), but there is one problem.

# DiagnosticSource and EventSource
There are two main classes to send traces `System.Diagnostics.DiagnosticSource` and `System.Diagnostics.Tracing.EventSource`. The first one is used for logging data within the process, so it allows non-serializable types. The second one, on the opposite, assumes that the events will leave the process and requires only serialisable data. As you may see, the `EventSource` class produces events that can be consumed via the `dotnet-trace` tool. The main problem is that MassTransit uses `DiagnosticSource`.

How can we deal with that?

There is one solution called [`DiagnosticSourceEventSource`](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/System/Diagnostics/DiagnosticSourceEventSource.cs). It is a bridge between `DiagnosticSource` and `EventSource`. With its help, we can consume events via the tracing tool from any `DiagnosticSource`.

Let's start our project. I'm using the `async-request-processing` application from the previous posts.

```
$ dotnet run
```

Now, we need to specify the provider for `dotnet-trace collect` command as `Microsoft-Diagnostics-DiagnosticSource` and an argument `FilterAndPayloadSpecs`. The argument contains an interested `DiagnosticSource`. Also, it may provide an event name, a list of transformations, etc. You can find the entire structure of the argument in the [source code](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/System/Diagnostics/DiagnosticSourceEventSource.cs).

```
$ dotnet-trace collect --process-id 184530 --providers=Microsoft-Diagnostics-DiagnosticSource:3:5:FilterAndPayloadSpecs=MassTransit
```

Finally, open the result in [PerfView](https://github.com/microsoft/perfview). The events are available for analysis.

![PerfView](/images/2021-09-22-tracing-for-messaging-application/perfview.jpg)

# Conclusion
In this short post, I've briefly touched on tracing and showed how to consume events with the `dotnet-trace` tool from `DiagnosticSource` class.

# References

- [Conversation about diagnostics](https://devblogs.microsoft.com/dotnet/conversation-about-diagnostics/)
- [dotnet-trace performance analysis utility](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
- [DiagnosticSource User's Guide](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/DiagnosticSourceUsersGuide.md)
- [DiagnosticSourceEventSource](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Diagnostics.DiagnosticSource/src/System/Diagnostics/DiagnosticSourceEventSource.cs)
- [MassTransit DiagnosticSource](https://masstransit-project.com/advanced/monitoring/diagnostic-source.html)
- [.NET Core logging and tracing](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/logging-tracing)

*Image: Photo by Anne Nygård on Unsplash*