---
title: "SynchronizationContext — Note 2"
excerpt: "In this post, I want to take a look at some existing examples of SynchronizationContext."
header:
  overlay_image: /assets/images/2022-10-17-synchronization-context-note-2/cover.jpg
  show_overlay_excerpt: false
  caption: "Photo by [Robert Katzki](https://unsplash.com/@ro_ka) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2022-10-17
tags: ["SynchronizationContext", "Task", "Thread", "TPL", "Async and Await"]
---
To get a better understanding, it’s always handy to look at a few examples.

# `WindowsFormsSynchronizationContext`

The first one
is [`WindowsFormsSynchronizationContext`](https://github.com/dotnet/winforms/blob/main/src/System.Windows.Forms/src/System/Windows/Forms/WindowsFormsSynchronizationContext.cs).
This context is used while working with a Windows Forms application. When you’re dealing with UI, you need to update
your controls from the main thread. So, this context schedules all work for one thread.

```csharp
public override void Send(SendOrPostCallback d, object state)
{
    Thread destinationThread = DestinationThread;
    if (destinationThread is null || !destinationThread.IsAlive)
    {
        throw new InvalidAsynchronousStateException(SR.ThreadNoLongerValid);
    }

    controlToSendTo?.Invoke(d, new object[] { state });
}

public override void Post(SendOrPostCallback d, object state)
{
    controlToSendTo?.BeginInvoke(d, new object[] { state });
}
```

The documentation
of [`Control.Invoke`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.control.invoke?view=windowsdesktop-6.0)
and [`Control.BeginInvoke`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.control.begininvoke?view=windowsdesktop-6.0)
says that

> `Invoke` - Executes a delegate on the thread that owns the control's underlying window handle.
`BeginInvoke` - Executes a delegate asynchronously on the thread that the control's underlying handle was created on.

Indeed, every callback will run on a single thread.

# `DispatcherSynchronizationContext`

Next context
is [`DispatcherSynchronizationContext`](https://github.com/dotnet/wpf/blob/main/src/Microsoft.DotNet.Wpf/src/WindowsBase/System/Windows/Threading/DispatcherSynchronizationContext.cs).
It is used by WPF applications, so it should also schedule callbacks to the main thread.

```csharp
internal Dispatcher _dispatcher;

public override void Send(SendOrPostCallback d, Object state)
{
    // Call the Invoke overload that preserves the behavior of passing
    // exceptions to Dispatcher.UnhandledException.  
    if(BaseCompatibilityPreferences.GetInlineDispatcherSynchronizationContextSend() && _dispatcher.CheckAccess())
    {
        // Same-thread, use send priority to avoid any reentrancy.
        _dispatcher.Invoke(DispatcherPriority.Send, d, state);
    }
    else
    {
        // Cross-thread, use the cached priority.
        _dispatcher.Invoke(_priority, d, state);
    }
}

public override void Post(SendOrPostCallback d, Object state)
{
    // Call BeginInvoke with the cached priority.  Note that BeginInvoke
    // preserves the behavior of passing exceptions to
    // Dispatcher.UnhandledException unlike InvokeAsync.  This is
    // desireable because there is no way to await the call to Post, so
    // exceptions are hard to observe.
    _dispatcher.BeginInvoke(_priority, d, state);
}
```

Here is a quote about `Dispatcher` class from
the [documentation](https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=windowsdesktop-6.0):

>
The [Dispatcher](https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=windowsdesktop-6.0)
maintains a prioritized queue of work items for a specific thread.
When
a [Dispatcher](https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=windowsdesktop-6.0)
is created on a thread, it becomes the
only [Dispatcher](https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=windowsdesktop-6.0)
that can be associated with the thread, even if
the [Dispatcher](https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher?view=windowsdesktop-6.0)
is shut down.

Thus, it is clear that in this case, too, all the work will be done on a single thread.

# `MaxConcurrencySyncContext`

Another interesting example is
a [context](https://github.com/xunit/xunit/blob/main/src/xunit.v3.core/Sdk/MaxConcurrencySyncContext.cs) from the xUnit
library. The main purpose of it is to limit the number of threads that will execute unit tests.

```csharp
public class MaxConcurrencySyncContext : SynchronizationContext, IDisposable
{
    readonly List<Thread> workerThreads;
    readonly ConcurrentQueue<(SendOrPostCallback callback, object? state, ExecutionContext? context)> workQueue = new();

    ...

    public MaxConcurrencySyncContext(int maximumConcurrencyLevel)
    {
        workerThreads =
            Enumerable
                .Range(0, maximumConcurrencyLevel)
                .Select(_ => { var result = new Thread(WorkerThreadProc); result.Start(); return result; })
                .ToList();
    }
    
    ...
}
```

This context creates a `workQueue` for working items and a bunch of threads to handle that queue.

And here is how the two main methods look like.

```csharp
public override void Post(SendOrPostCallback d, object? state)
{
    // HACK: DNX on Unix seems to be calling this after it's disposed. In that case,
    // we'll just execute the code directly, which is a violation of the contract
    // but should be safe in this situation.
    if (disposed)
        Send(d, state);
    else
    {
        var context = ExecutionContext.Capture();
        workQueue.Enqueue((d, state, context));
        workReady.Set();
    }
}

public override void Send(SendOrPostCallback d, object? state)
{
    d(state);
}
```

Of course, it's curious to try this context. I’ve created a small application:

```csharp
var maximumConcurrencyLevel = 1;
Console.WriteLine($"Current thread: {Environment.CurrentManagedThreadId}");
var syncContext = new MaxConcurrencySyncContext(maximumConcurrencyLevel);
for (var i = 0; i < 5; i++)
{
    syncContext.Post(_ => Console.WriteLine($"Thread id: {Environment.CurrentManagedThreadId}"), null);
}
Console.ReadKey();
```

By setting different values of the `maximumConcurrencyLevel` parameter, you can see how the result changes.

`var maximumConcurrencyLevel = 1;`

```
Current thread: 1
Thread id: 7
Thread id: 7
Thread id: 7
Thread id: 7
Thread id: 7
```

`var maximumConcurrencyLevel = 3;`

```
Current thread: 1
Thread id: 7
Thread id: 9
Thread id: 8
Thread id: 7
Thread id: 9
```

`var maximumConcurrencyLevel = 5;`

```
Current thread: 1
Thread id: 8
Thread id: 10
Thread id: 9
Thread id: 7
Thread id: 11
```

It seems like it's time to try to create my own context.

[*Next note*]({% post_url 2022-11-19-synchronization-context-note-3 %})

# References

*The source code is available [here](https://github.com/rafaelldi/asynchronous-playground/tree/main/synchronization-context-app)*

- [`WindowsFormsSynchronizationContext`](https://github.com/dotnet/winforms/blob/main/src/System.Windows.Forms/src/System/Windows/Forms/WindowsFormsSynchronizationContext.cs)
- [`DispatcherSynchronizationContext`](https://github.com/dotnet/wpf/blob/main/src/Microsoft.DotNet.Wpf/src/WindowsBase/System/Windows/Threading/DispatcherSynchronizationContext.cs)
- [WPF - Threading Model](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/threading-model)
- [`MaxConcurrencySyncContext`](https://github.com/xunit/xunit/blob/main/src/xunit.v3.core/Sdk/MaxConcurrencySyncContext.cs)