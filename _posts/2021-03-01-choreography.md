---
title: "Choreography"
categories: posts
author: Rival Abdrakhmanov
date: 2021-03-01
tags: ["Distributed application", "Choreography", "MassTransit", "Messaging", "ASP.NET Core"]
---
This post shows you an example of the choreography pattern to coordinate services in a system. 

![Title image](/images/2021-03-01-choreography/cover_choreography.jpg)

In the previous post, we created a food delivery application and applied the orchestration pattern. In this one, I'm going to modify that solution to follow the choreography pattern.

[Orchestration](/posts/orchestration.html)

Let me remind the process:
1. A user places online order from the website;
2. The manager receives a notification about the new order and accepts (or denies) it;
3. The kitchen gets the order details and starts cooking;
4. The courier delivers food to the user's address.

![Food delivery schema](/images/2021-03-01-choreography/food-delivery.png)

# Routing slip

Sometimes you don't want to couple to one central service (orchestrator) as we did in the previous post. In that case, it's possible to connect your services via pub/sub mechanism. It's not so complicated and has some pitfalls, especially if you want to add compensating transactions. So, today I'll additionally apply a routing slip pattern. I described it in the post about coordination. In short, you put all steps (and compensations if needed) in a routing slip and attach it to the message. Hence, every service knows where to send this message next.

[Coordination in the distributed systems](/posts/coordination-in-the-distributed-systems.html)

# Modify application

Notice that after the `Place Order` step, our pipeline stops and waits for the manager reaction. So, we won't include this step in the routing slip.

Let's modify the `PlaceOrder` method in the `OrdersController`. We need to send `OrderPlaced` event to reuse the corresponding consumer, which will send a notification to the manager. Also, we save order details and address to cash because we'll need them in the `AcceptOrder` method.

```c#
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IBus _bus;
    private static readonly Dictionary<Guid, (string OrderDetails, string Address)> Cash = new();

    public OrdersController(IBus bus)
    {
        _bus = bus;
    }

    [HttpPost]
    public async Task<IActionResult> PlaceOrder(OrderDto dto)
    {
        var orderId = Guid.NewGuid();

        Cash.Add(orderId, (dto.OrderDetails, dto.Address));

        await _bus.Publish(new OrderPlaced
        {
            OrderId = orderId,
            OrderDetails = dto.OrderDetails
        });

        return Ok();
    }

    // ...
}
```

That's all with the first step.

Next, to build a routing slip, we need to create an activity for each step in the pipeline. So, we end with `CookDishActivity` and `DeliverOrderActivity`. They will send messages to the consumers (again, to reuse functionality from the previous part) and wait for the responses from them. After the activity is done, we call `context.Completed()` method to advance the routing slip to the next one. Each activity is located in the related service.

```c#
public class CookDishActivity : IExecuteActivity<CookDishArgument>
{
    private readonly IBus _bus;

    public CookDishActivity(IBus bus)
    {
        _bus = bus;
    }

    public async Task<ExecutionResult> Execute(ExecuteContext<CookDishArgument> context)
    {
        var client = context.CreateRequestClient<CookDish>(_bus);
        await client.GetResponse<DishCooked>(new CookDish
        {
            OrderId = context.Arguments.OrderId,
            OrderDetails = context.Arguments.OrderDetails
        });
        return context.Completed();
    }
}
```
```c#
public class DeliverOrderActivity : IExecuteActivity<DeliverOrderArgument>
{
    private readonly IBus _bus;
    
    public DeliverOrderActivity(IBus bus)
    {
        _bus = bus;
    }

    public async Task<ExecutionResult> Execute(ExecuteContext<DeliverOrderArgument> context)
    {
        var client = context.CreateRequestClient<DeliverOrder>(_bus);
        await client.GetResponse<OrderDelivered>(new DeliverOrder
        {
            OrderId = context.Arguments.OrderId,
            Address = context.Arguments.Address
        });
        return context.Completed();
    }
}
```

After that, register them in the `Startup`.

```c#
services.AddMassTransit(x =>
    {
        // ...

        x.AddExecuteActivity<CookDishActivity, CookDishArgument>();
        x.AddExecuteActivity<DeliverOrderActivity, DeliverOrderArgument>();

        //...
    })
```

Finally, build the routing slip. As you see, we don't include any message types or consumer details. All we need are the addresses of activities. Therefore, we reduce coupling between components in the system. As I said early, this routing slip will be attached to the message, and each service will know where to send it next.

```c#
[HttpPost("{id}/accept")]
public async Task<IActionResult> AcceptOrder(Guid id)
{
    if (!Cash.TryGetValue(id, out var order))
    {
        throw new ArgumentException("Can't find order details");
    }

    var builder = new RoutingSlipBuilder(Guid.NewGuid());

    builder.AddActivity("CookDish", new Uri("queue:CookDish_execute"), new CookDishArgument
    {
        OrderId = id,
        OrderDetails = order.OrderDetails
    });

    builder.AddActivity("DeliverOrder", new Uri("queue:DeliverOrder_execute"), new DeliverOrderArgument
    {
        OrderId = id,
        Address = order.Address
    });

    var routingSlip = builder.Build();

    await _bus.Execute(routingSlip);
    return Ok();
}
```

Our choreography-based system is ready. We've removed the central component, which instructs others what to do and oversees the process. Now, we initially create instructions and send them with a message.

If you test the application, you'll see similar logs.

```
info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/1.1 POST http://localhost:5000/choreography/orders application/json 62
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP/1.1 POST http://localhost:5000/choreography/orders application/json 62 - 200 0 - 393.6161ms
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      OrderPlaced event received
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Order with id = 98641c8b-4ec5-4859-a0a1-8732b4ef600b and details = Pizza was placed
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Sending notification to the manager...

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/1.1 POST http://localhost:5000/orders/98641c8b-4ec5-4859-a0a1-8732b4ef600b/accept application/json 3
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP/1.1 POST http://localhost:5000/orders/98641c8b-4ec5-4859-a0a1-8732b4ef600b/accept application/json 3 - 200 0 - 70.4028ms
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      CookDish command received
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      Dish for order with id = 98641c8b-4ec5-4859-a0a1-8732b4ef600b was cooked
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      DeliverOrder command received
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      Order with id = 98641c8b-4ec5-4859-a0a1-8732b4ef600b was delivered
```

All code is available on GitHub. You can compare two approaches side-by-side.

[Link to GitHub Project](https://github.com/rafaelldi/communication-food-delivery)

# Conclusion

Today, we've transformed the application with a choreography approach to reduce coupling and increase the autonomy of components in the system. Each pattern has advantages and disadvantages. It depends on a lot of factors which one you want to apply. I hope these posts give you some ideas about how to coordinate services in a distributed system.

# References

* [Coordination in the distributed systems](/posts/coordination-in-the-distributed-systems.html)
* https://masstransit-project.com/
* https://masstransit-project.com/advanced/courier/

*Image: Photo by Gaelle Marcel on Unsplash*