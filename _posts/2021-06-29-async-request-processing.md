---
title: "Async request processing"
excerpt: "Recently, I've talked about state machines and routing slips. In this post, I am going to show how to combine these approaches."
header:
  og_image: /images/2021-06-29-async-request-processing/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2021-06-29
tags: ["Distributed application", "MassTransit", "Messaging", "ASP.NET Core", "State Machine", "Routing Slip"]
---

![Title image](/images/2021-06-29-async-request-processing/cover.jpg)

This post came out of a real-life scenario. Imagine that you have a list of items, and you can somehow process each of them. For example, a list of pull requests on GitHub, every one of them you can automatically review. Another key feature is that each operation can take a long time. So, you don't want to perform them synchronously.

As in the previous posts, let's start with a simple web application and add [MassTransit](https://masstransit-project.com/) packages.

```
$ dotnet new web
$ dotnet add package MassTransit.AspNetCore
$ dotnet add package MassTransit.RabbitMQ
```

Next, I need some mechanism to process each element separately, one by one. I don't know how many items will be in a request, so I have to add them dynamically. I think that the routing slip will be appropriate for our case.

I don't go much into details about implementing routing slip (and saga) in MassTransit because I have separate posts about them. And the library has perfect documentation.

I've created a simple activity that logs some information and pauses for a random amount of time. After the delay, it publishes a message about completed item processing.

```c#
public record ProcessItemArgument(Guid ItemId);
public record ItemProcessed(Guid Id, Guid TrackingNumber);

public class ProcessItemActivity : IExecuteActivity<ProcessItemArgument>
{
    private readonly RandomService _randomService;
    private readonly ILogger<ProcessItemActivity> _logger;

    public ProcessItemActivity(RandomService randomService, ILogger<ProcessItemActivity> logger)
    {
        _randomService = randomService;
        _logger = logger;
    }

    public async Task<ExecutionResult> Execute(ExecuteContext<ProcessItemArgument> context)
    {
        _logger.LogInformation("Processing item with id={ItemId}", context.Arguments.ItemId);

        var delay = _randomService.GetDelay();
        await Task.Delay(TimeSpan.FromSeconds(delay));

        await context.Publish(new ItemProcessed(context.Arguments.ItemId, context.TrackingNumber));
        
        return context.Completed();
    }
}
```

The next step is to build the routing slip with these activities. I will use a state machine to save the status of the process and check it via http request. When a command to process items arrives, we need to create the routing slip, go to the `Processing` state and respond that we accepted the request. Also, we save into the state all ids that we need to process.

```c#
Initially(
    When(ProcessRequestCommand)
        .Then(context =>
        {
            logger.LogInformation("Start processing request with id={RequestId}", context.Data.RequestId);
            context.Instance.RequestId = context.Data.RequestId;
            context.Instance.ToProcess.AddRange(context.Data.ItemsIds);
        })
        .CreateRoutingSlip()
        .TransitionTo(Processing)
        .Respond(context => new RequestAccepted(context.Instance.RequestId)));
```
```c#
public static EventActivityBinder<RequestProcessingState, ProcessRequestCommand> CreateRoutingSlip(
    this EventActivityBinder<RequestProcessingState, ProcessRequestCommand> binder)
{
    return binder.ThenAsync(async context =>
    {
        var trackingNumber = Guid.NewGuid();
        context.Instance.TrackingNumber = trackingNumber;
        var builder = new RoutingSlipBuilder(trackingNumber);
        var consumeContext = context.CreateConsumeContext();

        foreach (var item in context.Data.ItemsIds)
        {
            builder.AddActivity("ProcessItem", new Uri("queue:ProcessItem_execute"), new ProcessItemArgument(item));
        }

        var routingSlip = builder.Build();
        await consumeContext.Execute(routingSlip);
    });
}
```

After an item has been processed, we need to update the state.

```c#
During(Processing,
    When(ItemProcessed)
        .Then(context =>
        {
            logger.LogInformation("Item with id={ItemId} is processed", context.Data.Id);
            context.Instance.ToProcess.Remove(context.Data.Id);
            context.Instance.Processed.Add(context.Data.Id);
        }));
```

Finally, after the routing slip is completed, the state machine goes to the Completed state.

```c#
During(Processing,
    When(RequestProcessingCompleted)
        .Then(context =>
        {
            logger.LogInformation("Process completed for request with id={RequestId}",
                context.Instance.RequestId);
        })
        .TransitionTo(Completed));
```

Our command side is ready; let's take a look at query one. How to monitor the process? I'm going to add a new message and handle it within the state machine.

```c#
public record RequestStatusQuery(Guid RequestId);
public record RequestStatus(string Status, List<Guid> ToProcess, List<Guid> Processed);

During(Processing,
    When(RequestStatusQueried)
        .Respond(context => new RequestStatus(
            $"{nameof(Processing)}",
            context.Instance.ToProcess,
            context.Instance.Processed
        )));

During(Completed,
    When(RequestStatusQueried)
        .Respond(context => new RequestStatus(
            $"{nameof(Completed)}",
            context.Instance.ToProcess,
            context.Instance.Processed
        )));
```

As you can see, we answer with a state and a number of the completed items. So, now let's create a controller to send messages to the state machine.

```c#
public record Request(Guid Id, List<Guid> Items);

[ApiController]
[Route("requests")]
public class RequestsController : ControllerBase
{
    private readonly IRequestClient<ProcessRequestCommand> _processRequestClient;
    private readonly IRequestClient<RequestStatusQuery> _getRequestStatusClient;

    public RequestsController(
        IRequestClient<ProcessRequestCommand> processRequestClient,
        IRequestClient<RequestStatusQuery> getRequestStatusClient)
    {
        _processRequestClient = processRequestClient;
        _getRequestStatusClient = getRequestStatusClient;
    }

    [HttpPost]
    public async Task<IActionResult> Process(Request request)
    {
        var response = await _processRequestClient.GetResponse<RequestAccepted>(new ProcessRequestCommand(request.Id, request.Items));
        return Accepted(response.Message.RequestId);
    }

    [HttpGet("{id:guid}/status")]
    public async Task<IActionResult> Status(Guid id)
    {
        var response = await _getRequestStatusClient.GetResponse<RequestStatus>(new RequestStatusQuery(id));
        return Ok(new
        {
            response.Message.Status,
            toProcess = string.Join(",", response.Message.ToProcess),
            processed = string.Join(",", response.Message.Processed)
        });
    }
}
```

This way, we can send a command and monitor the process by polling the status.

```http
POST http://localhost:5000/requests
Content-Type: application/json

{
  "id": "B41DEB0B-D088-458C-9241-53375B12117D",
  "items": [
    "BF1FAE17-070A-473D-9866-C20E594A4CB1",
    "8668F382-25B2-4C08-AB02-84B99F9BBCB6",
    "74A7C363-AF80-4EA6-83BF-CD415ADD7F0C"
  ]
}
```
```http
GET http://localhost:5000/requests/B41DEB0B-D088-458C-9241-53375B12117D/status

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Server: Kestrel
Transfer-Encoding: chunked

{
  "status": "Processing",
  "toProcess": "8668f382-25b2-4c08-ab02-84b99f9bbcb6,74a7c363-af80-4ea6-83bf-cd415add7f0c",
  "processed": "bf1fae17-070a-473d-9866-c20e594a4cb1"
}
```

You can find the project in [my repository](https://github.com/rafaelldi/async-request-processing).

# Conclusion
This post shows how to combine state machines and routing slips to handle requests with multiple items asynchronously.

# References
* [MassTransit](https://masstransit-project.com/)
* [State machines]({% post_url 2021-02-18-orchestration %})
* [Routing slips]({% post_url 2021-03-01-choreography %})

*Image: Photo by Marvin Castelino on Unsplash*