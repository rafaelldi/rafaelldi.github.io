---
title: "SynchronizationContext — Note 1"
excerpt: "In this post, I start to explore SynchronizationContext."
header:
overlay_image: /images/2022-10-16-synchronization-context-note-1/cover.jpg
show_overlay_excerpt: false
caption: "Photo by [Ashley Richards](https://unsplash.com/@docaah59) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2022-10-16
tags: ["SynchronizationContext", "Task", "Thread", "TPL", "Async and Await"]
---
I’ve decided to learn and take a closer look at some not-so-famous things in C#. I have knowledge about different
aspects of the language, and I want to recall them, systematize them, make little notes and of course, learn something
new. And my first stop is `SynchronizationContext`.

As far as I understand now, `SynchronizationContext` is a mechanism to group some threads and give them some jobs to
execute. For example, you can dedicate several threads specifically to IO operations. Or give to some library only a
limited set of threads. With `SynchronizationContext` consumer doesn't care about threads or how to manage them. He
schedules all his tasks with the help of this abstraction. We can switch the implementation, but for the consumer
everything will be the same.

This type has two main methods to schedule a work item: `Send` and `Post`. The first is for synchronous execution, the
second is for asynchronous execution. In the default context, they look as follows:

```csharp
public virtual void Send(SendOrPostCallback d, object? state) => d(state);
public virtual void Post(SendOrPostCallback d, object? state) => ThreadPool.QueueUserWorkItem(static s => s.d(s.state), (d, state), preferLocal: false);
```

To see this in action, I’ve created a small sample:

```csharp
Console.WriteLine($"Current thread: {Environment.CurrentManagedThreadId}");

var syncContext = new SynchronizationContext();
syncContext.Send(_ => Console.WriteLine($"Thread from the Send method: {Environment.CurrentManagedThreadId}"), null);
syncContext.Post(_ => Console.WriteLine($"Thread from the Pend method: {Environment.CurrentManagedThreadId}"), null);

Console.ReadKey();
```

And the result of this code:

```
Current thread: 1
Thread from the Send method: 1
Thread from the Pend method: 6
```

Thus, it could be said that default `SynchronizationContext` groups threads from the `ThreadPool` and schedules
asynchronous jobs on them.

[*Next note*]({% post_url 2022-10-17-synchronization-context-note-2 %})

# References

- [Parallel Computing - It's All About the SynchronizationContext](https://learn.microsoft.com/en-us/archive/msdn-magazine/2011/february/msdn-magazine-parallel-computing-it-s-all-about-the-synchronizationcontext)