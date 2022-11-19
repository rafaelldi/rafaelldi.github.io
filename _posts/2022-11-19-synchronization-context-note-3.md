---
title: "SynchronizationContext — Note 3"
excerpt: "In this post, I’m creating a simple example of SynchronizationContext."
header:
overlay_image: /images/2022-11-19-synchronization-context-note-3/cover.jpg
show_overlay_excerpt: false
caption: "Photo by [Jack Anstey](https://unsplash.com/@jack_anstey) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2022-11-19
tags: ["SynchronizationContext", "Task", "Thread", "TPL", "Async and Await"]
---

# `SimpleSynchronizationContext`

Let’s try to create a simple `SynchronizationContext`. I won’t implement a `Send` method because it is not so
interesting.

As you can see from the [`MaxConcurrencySyncContext`]({% post_url 2022-10-17-synchronization-context-note-2 %}) example,
we need a kind of task queue for the `Post` method. I will also create a separate thread, which purpose will be to
execute the tasks from this queue. As I said in the [first note]({% post_url 2022-10-16-synchronization-context-note-1 %}), 
it means that our synchronization context will encapsulate this thread.
Consumers don’t know how exactly their tasks will be handled. They just post the tasks and forget about them.

So, this is a very basic implementation of `SynchronizationContext`.

```csharp
public class SimpleSynchronizationContext : SynchronizationContext
{
    private readonly ConcurrentQueue<MyTask> _queue;
    private readonly Thread _processingThread;

    public SimpleSynchronizationContext()
    {
        _queue = new ConcurrentQueue<MyTask>();
        _processingThread = new Thread(Handle);
        _processingThread.Start();
    }

    private void Handle()
    {
        Console.WriteLine($"Start processing tasks on thread: {Environment.CurrentManagedThreadId}");

        var spinWait = new SpinWait();
        while (true)
        {
            if (_queue.TryDequeue(out var myTask))
            {
                myTask.Callback(myTask.State);
            }
            else
            {
                spinWait.SpinOnce();
            }
        }
    }

    public override void Send(SendOrPostCallback d, object? state)
    {
        throw new NotImplementedException();
    }

    public override void Post(SendOrPostCallback d, object? state)
    {
        var myTask = new MyTask(d, state);
        _queue.Enqueue(myTask);
    }
}

public readonly record struct MyTask(SendOrPostCallback Callback, object? State);
```

Let’s try it in action.

```csharp
static void RunWithSimpleContext()
{
    Console.WriteLine($"Current thread: {Environment.CurrentManagedThreadId}");

    var syncContext = new SimpleSynchronizationContext();
    for (var i = 0; i < 5; i++)
    {
        syncContext.Post(_ => Console.WriteLine($"Thread id: {Environment.CurrentManagedThreadId}"), null);
    }

    Console.ReadKey();
}
```

```
Current thread: 1
Start processing tasks on thread: 7
Thread id: 7
Thread id: 7
Thread id: 7
Thread id: 7
Thread id: 7
```

# `AffinitySynchronizationContext`

Another fun example is to execute all tasks in a given core of your processor. It’s similar to the previous one, but I
use the `IdealProcessor` and `ProcessorAffinity` properties to set up a specified core.

```csharp
public class AffinitySynchronizationContext : SynchronizationContext
{
    private readonly int _processorId;
    private readonly ConcurrentQueue<MyTask> _queue;
    private readonly Thread _processingThread;
    
    public AffinitySynchronizationContext(int processorId)
    {
        _processorId = processorId;
        _queue = new ConcurrentQueue<MyTask>();
        _processingThread = new Thread(Handle)
        {
            IsBackground = true
        };
        _processingThread.Start();
    }

    private void Handle()
    {
        var threadId = Util.GetCurrentThreadId();
        var process = Process.GetCurrentProcess();
        var thread = process.Threads.OfType<ProcessThread>().Single(it => it.Id == threadId);
        thread.IdealProcessor = _processorId;
        thread.ProcessorAffinity = 1 << _processorId;
        
        var spinWait = new SpinWait();
        while (true)
        {
            if (_queue.TryDequeue(out var myTask))
            {
                myTask.Callback(myTask.State);
            }
            else
            {
                spinWait.SpinOnce();
            }
        }
    }
    
    public override void Send(SendOrPostCallback d, object? state)
    {
        throw new NotImplementedException();
    }

    public override void Post(SendOrPostCallback d, object? state)
    {
        var myTask = new MyTask(d, state);
        _queue.Enqueue(myTask);
    }
}
```

After that, we need to schedule some work for that context.

```csharp
static void RunWithAffinityContext()
{
    Console.WriteLine("Star");

    var syncContext = new AffinitySynchronizationContext(8);
    for (var i = 0; i < 500_000; i++)
    {
        syncContext.Post(
            _ =>
            {
                long t = 0;
                for (var i = 0; i < 20_000; i++)
                {
                    t += i / 42 - i * 3;
                }
            },
            null
        );
    }
    
    Console.WriteLine("Finished");
    Console.ReadKey();
}
```

If you open a system monitor, you’ll see that only one of the cores is busy.

![System monitor](/images/2022-11-19-synchronization-context-note-3/sysmon.png)