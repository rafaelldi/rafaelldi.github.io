---
layout: post
title: "Orchestration"
categories: misc
---
In this post, I want to show you an example of how to apply an orchestration pattern in your system. 

![Title image](https://raw.githubusercontent.com/rafaelldi/rafaelldi.github.io/master/images/2021-02-18-Orchestration/cover_orchestration.jpg)

The previous post was about two main patterns for coordinating services in a distributed system. So, today we'll take a look at the straightforward application for food delivery. Within this system, we'll try to connect different parts with orchestration pattern.

[Coordination in the distributed systems](https://northern-dev.net/coordination-in-the-distributed-systems/)

_Spoiler: this post has a lot of code. If you want to look at the project yourself, I left a link to GitHub at the end of the post._

Let's pretend we're developing an application for the restaurant. As you may notice, food delivery is popular nowadays, so we want to implement this functionality in the app.

![Food delivery schema](https://raw.githubusercontent.com/rafaelldi/rafaelldi.github.io/master/images/2021-02-18-Orchestration/food-delivery.png)

The whole process will be consists of four steps:
1. A user places online order from the website; 
2. The manager receives a notification about the new order and accepts (or denies) it;
3. The kitchen gets the order details and starts cooking;
4. The courier delivers food to the user's address.

# Application

Let's start with an empty ASP.NET template.

```
$ dotnet new web
```

We'll use an excellent library MassTransit, so we need to install this package.

```
$ dotnet add package MassTransit.AspNetCore
```

[MassTransit](https://masstransit-project.com/)

# Controller and consumers

First of all, I'm going to add a new controller with two methods `PlaceOrder` and `AcceptOrder`. The first one will be used by customers and the second one by managers. Also, we need to register the controller in the `Startup` class.

```c#
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> PlaceOrder(OrderDto dto)
    {
        return Ok();
    }

    [HttpPost("{id}/accept")]
    public async Task<IActionResult> AcceptOrder(Guid id)
    {
        return Ok();
    }
}
```
```c#
public record OrderDto(string OrderDetails, string Address);
```
```c#
public class Startup
 {
     public void ConfigureServices(IServiceCollection services)
     {
         services.AddControllers();
     }

     public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
     {
         app.UseRouting();

         app.UseEndpoints(endpoints =>
         {
             endpoints.MapControllers();
         });
     }
 }
```

Now, create commands and events. We'll use them to trigger steps from our pipeline. You can easily relate them to the diagram above. Commands are sent to perform some action, events to report that the action has happened.

```c#
public static class Commands
{
    public record PlaceOrder
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
        public string Address { get; init; }
    }

    public record AcceptOrder
    {
        public Guid OrderId { get; init; }
    }

    public record CookDish
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
    }

    public record DeliverOrder
    {
        public Guid OrderId { get; init; }
        public string Address { get; init; }
    }
}
```
```c#
public static class Events
{
    public record OrderPlaced
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
    }

    public record DishCooked
    {
        public Guid OrderId { get; init; }
    }

    public record OrderDelivered
    {
        public Guid OrderId { get; init; }
    }
}
```

After that, let's add consumers. Consumers are similar to controllers but used for messaging. Our consumers are straightforward; they just log some information. Of course, in the real application, logic is more complicated. Moreover, they might be located in different microservices, but I'll create them in our API project for simplicity. I recently showed the approach with multiple containers connected via RabbitMQ.

[Distributed application with Project Tye](https://northern-dev.net/distributed-application-with-project-tye/)

```c#
public class OrderPlacedConsumer : IConsumer<OrderPlaced>
{
    private readonly ILogger<OrderPlacedConsumer> _logger;

    public OrderPlacedConsumer(ILogger<OrderPlacedConsumer> logger)
    {
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderPlaced> context)
    {
        await Task.Delay(500);

        _logger.LogInformation("Order with id = {id} and details = {details} was placed",
            context.Message.OrderId.ToString(), context.Message.OrderDetails);
        _logger.LogInformation("Sending notification to the manager...");
    }
}
```
```c#
public class CookDishConsumer : IConsumer<CookDish>
{
    private readonly ILogger<CookDishConsumer> _logger;

    public CookDishConsumer(ILogger<CookDishConsumer> logger)
    {
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<CookDish> context)
    {
        await Task.Delay(500);

        var orderId = context.Message.OrderId;
        _logger.LogInformation("Dish for order with id = {id} was cooked", orderId.ToString());
        await context.Publish(new DishCooked {OrderId = orderId});
    }
}
```
```c#
public class DeliverOrderConsumer : IConsumer<DeliverOrder>
{
    private readonly ILogger<DeliverOrderConsumer> _logger;

    public DeliverOrderConsumer(ILogger<DeliverOrderConsumer> logger)
    {
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<DeliverOrder> context)
    {
        await Task.Delay(500);

        var orderId = context.Message.OrderId;
        _logger.LogInformation("Order with id = {id} was delivered", orderId.ToString());
        await context.Publish(new OrderDelivered {OrderId = orderId});
    }
}
```

Notice that consumers perform their actions in response to the messages they receive. 

Possible implementations of consumers in different services are shown in the diagram below.

![Multiple services schema](https://raw.githubusercontent.com/rafaelldi/rafaelldi.github.io/master/images/2021-02-18-Orchestration/food-delivery-services.png)

Next, register the consumers and the library itself. For testing purposes, I'm using in-memory message bus.

```c#
services.AddMassTransit(x =>
    {
        x.AddConsumer<CookDishConsumer>();
        x.AddConsumer<OrderPlacedConsumer>();
        x.AddConsumer<DeliverOrderConsumer>();

        x.UsingInMemory((context, cfg) => { cfg.ConfigureEndpoints(context); });
    })
    .AddMassTransitHostedService();
```

Now, all groundwork is done, let's connect different parts.

# State machine

The heart of our system is an orchestrator, which will be inside the API project. It will follow the business process, keep the state and send messages to other components. It's natural to implement our pipeline as a state machine. MassTransit gives us a wide range of possibilities to customize it; we'll use only a few of them. Here you can find the documentation about state machines in MassTransit.

[Automatonymous](https://masstransit-project.com/usage/sagas/automatonymous.html)

Firstly, let's describe a state that we'll keep for each instance of the state machine.

```c#
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public int CurrentState { get; set; }

    public Guid OrderId { get; set; }
    public string OrderDetails { get; set; }
    public string Address { get; set; }
    public DateTime Placed { get; set; }
    public DateTime Accepted { get; set; }
    public DateTime Cooked { get; set; }
    public DateTime Delivered { get; set; }
}
```

`CorrelationId` and `CurrentState` are required fields; `OrderId`, `OrderDetails` and `Address` come from user request; `Placed`, `Accepted`, `Cooked` and `Delivered` fields we'll use to save the time of the events.

Next, create a state machine, add possible states and events.

```c#
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState, Placed, Accepted, Cooked);
    }

    public Event<PlaceOrder> PlaceOrder { get; private set; }
    public Event<AcceptOrder> AcceptOrder { get; private set; }
    public Event<DishCooked> DishCooked { get; private set; }
    public Event<OrderDelivered> OrderDelivered { get; private set; }

    public State Placed { get; private set; }
    public State Accepted { get; private set; }
    public State Cooked { get; private set; }
}
```

After that, we should specify the fields to correlate our messages. It tells a state machine which instance should be chosen to apply incoming event.

```c#
public OrderStateMachine()
{
    // ...

    Event(() => PlaceOrder, x => x.CorrelateById(m => m.Message.OrderId));
    Event(() => AcceptOrder, x => x.CorrelateById(m => m.Message.OrderId));
    Event(() => DishCooked, x => x.CorrelateById(m => m.Message.OrderId));
    Event(() => OrderDelivered, x => x.CorrelateById(m => m.Message.OrderId));
}
```

Then, add reactions to the incoming messages. You can see that I describe how to change states, publish other messages and save something to the instance of the state machine.

```c#
public OrderStateMachine()
{
    // ...

    Initially(
        When(PlaceOrder)
            .SetOrderDetails()
            .TransitionTo(Placed)
            .PublishOrderPlaced());

    During(Placed,
        When(AcceptOrder)
            .SetAcceptedTime()
            .TransitionTo(Accepted)
            .PublishCookDish());

    During(Accepted,
        When(DishCooked)
            .SetCookedTime()
            .TransitionTo(Cooked)
            .PublishDeliverOrder());

    During(Cooked,
        When(OrderDelivered)
            .SetDeliveredTime()
            .Finalize());
}

// ...

public static EventActivityBinder<OrderState, PlaceOrder> SetOrderDetails(
    this EventActivityBinder<OrderState, PlaceOrder> binder)
{
    return binder.Then(x =>
    {
        x.Instance.OrderId = x.Data.OrderId;
        x.Instance.OrderDetails = x.Data.OrderDetails;
        x.Instance.Address = x.Data.Address;
        x.Instance.Placed = DateTime.UtcNow;
    });
}

public static EventActivityBinder<OrderState, PlaceOrder> PublishOrderPlaced(
    this EventActivityBinder<OrderState, PlaceOrder> binder)
{
    return binder.PublishAsync(context => context.Init<OrderPlaced>(new OrderPlaced
    {
        OrderId = context.Data.OrderId,
        OrderDetails = context.Data.OrderDetails
    }));
}

// ...
```

Eventually, modify controllers to send corresponding commands and register state machine in the `Startup` class.

```c#
// ...
private readonly IPublishEndpoint _publishEndpoint;

public OrdersOrchestrationController(IPublishEndpoint publishEndpoint)
{
    _publishEndpoint = publishEndpoint;
}

[HttpPost]
public async Task<IActionResult> PlaceOrder(OrderDto dto)
{
    await _publishEndpoint.Publish(new PlaceOrder
    {
        OrderId = Guid.NewGuid(),
        OrderDetails = dto.OrderDetails,
        Address = dto.Address
    });
    return Ok();
}

[HttpPost("{id}/accept")]
public async Task<IActionResult> AcceptOrder(Guid id)
{
    await _publishEndpoint.Publish(new AcceptOrder
    {
        OrderId = id
    });
    return Ok();
}
```
```c#
x.AddSagaStateMachine<OrderStateMachine, OrderState>()
    .InMemoryRepository();
```

The whole process is as follows. The user sends a request to create an order. The controller transforms this request into a `PlaceOrder` command and sends it to the message bus. The state machine receives the command, sets itself to `Placed` status and sends the `OrderPlaced` event. The corresponding `OrderPlacedConsumer` responds to this event and sends a notification to the manager about the new order. At this point, the state machine pauses and waits for action from the user. After the manager approves the order with a request, the controller sends an `AcceptOrder` command. The state machine responds, sends a `CookDish` command, waits for a message from the `DishCookedConsumer` and sends a `DeliverOrder` command to deliver the order. After the order is delivered, the message `OrderDelivered` comes, the state machine passes to the final state.

After all this work, you can finally test our project. Send `POST` request to the controller and follow all the steps of our process. In the logs you will see the next entries.

```
info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/1.1 POST http://localhost:5000/orchestration/orders application/json 55
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Order with id = 99271053-7720-43de-a2fa-74a7b4d61e48 and details = Burger was placed
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Sending notification to the manager...

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/1.1 POST http://localhost:5000/orchestration/orders/99271053-7720-43de-a2fa-74a7b4d61e48/accept application/json 3
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      Dish for order with id = 99271053-7720-43de-a2fa-74a7b4d61e48 was cooked
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      Order with id = 99271053-7720-43de-a2fa-74a7b4d61e48 was delivered
```

It is worth noting that you don't wait for the process to complete inside the controller (like we did) in real systems. Mostly, long-running processes are handled asynchronously. I'm going to show you how to do it in future posts.

I didn't show all the code because the post is long enough as it is. You can find the project on GitHub.

[Link to GitHub Project](https://github.com/rafaelldi/communication-food-delivery)

# Conclusion

In this post, I've shown a possible implementation of the orchestration pattern. We built the application for food delivery which have the orchestrator and three consumers. I didn't separate them into different services because the example is already long. But with MassTransit, it's straightforward to do that. If there is too much code for you, you can easily download the project from GitHub and explore it by yourself. If you have any question, I will be happy to answer them in the comments below.

In the next post, I will show how to modify this solution towards the choreography pattern. I hope it will contain a much smaller amount of code 😉.

# References

* https://northern-dev.net/coordination-in-the-distributed-systems/
* https://masstransit-project.com/
* https://masstransit-project.com/usage/sagas/automatonymous.html

Image: Photo by Andrey Konstantinov on Unsplash