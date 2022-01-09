---
title: "ASP.NET Core: Health checks"
excerpt: "I'm starting a new series about application observability, and the first point is health checks. In this post, I describe how to add them to your ASP.NET Core application and monitor them."
header:
  og_image: /images/2019-05-06-asp-net-core-health-checks/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2019-05-06
tags: ["Observability", "ASP.NET Core", "Health Checks"]
---

![Title image](/images/2019-05-06-asp-net-core-health-checks/cover.jpg)

If you're using Kubernetes or load balancer, you need a tool, which can check that the application is still running. For these purposes, health checks provide an endpoint with the status of your application.

I've created a [simple project](https://github.com/rafaelldi/AspNetCoreAppHealthCheck) on GitHub with examples of this post.

# Health checks in ASP.NET Core 2.2
In this version of the framework, we have a built-in middleware provided health checks functionality. You need to put some lines in your Startup class:

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseHealthChecks("/healthcheck");
    }
}
```

Let's make a [simple request](https://github.com/rafaelldi/AspNetCoreAppHealthCheck/blob/master/requests.http) to `http://localhost:5000/healthcheck` and check the result:

![Health check request](/images/2019-05-06-asp-net-core-health-checks/health-check-request.png)

We see our application is `Healthy` and response code is `200(OK)`.

# Health checks UI
You can see application status not only by http-request, but it is also simple to add UI. You need to install [this nuget-package](https://www.nuget.org/packages/AspNetCore.HealthChecks.UI/) and make some changes in the code:

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks();
    services.AddHealthChecksUI();
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseHealthChecks("/healthcheck", new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    app.UseHealthChecksUI();
}
```
```json
"HealthChecks-UI": {
  "HealthChecks": [
    {
      "Name": "HTTP-Api-Basic",
      "Uri": "http://localhost:5000/healthcheck"
    }
  ],
  "EvaluationTimeOnSeconds": 60,
  "MinimumSecondsBetweenFailureNotifications": 60
}
```
After that, we will have this mini dashboard on `http://localhost:5000/healthchecks-ui`:

![Health dashboard](/images/2019-05-06-asp-net-core-health-checks/health-dashboard.png)

**Update**: There are some changes in .NET Core 3.0, which are caused by new routing system. You can find more details in the [official documentation](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.0#use-health-checks-routing).

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks();
        services.AddHealthChecksUI();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapHealthChecks("/health", new HealthCheckOptions
            {
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });
        });
        app.UseHealthChecksUI();
    }
}
```

# Database health check
Let's assume that your application has a reference to a third-party resource, for example, MongoDB and you want to know the status of this connection. Add [this package](https://www.nuget.org/packages/AspNetCore.HealthChecks.MongoDb/) to the project and add a method `AddMongoDb(...)`:

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddMongoDb(Configuration["MongoConnection:ConnectionString"]);
    services.AddHealthChecksUI();
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseHealthChecks("/healthcheck", new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    app.UseHealthChecksUI();
}
```

Now you can see a new item in the dashboard.

![Health dashboard with database](/images/2019-05-06-asp-net-core-health-checks/db-health-dashboard.png)

In [this repository](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks), you can find more packages for different databases and services. Furthermore, there is a [package](https://www.nuget.org/packages/Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore/) for the EF Core.

# Create your own health check
What if you have a specific logic for a health check? Then you need to realize the `IHealthCheck` interface. Let's consider the [example](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-2.2#create-health-checks) from Microsoft documentation:

```c#
public class ExampleHealthCheck : IHealthCheck
{
    public ExampleHealthCheck()
    {
    }

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default)
    {
        var healthCheckResultHealthy = true;

        return healthCheckResultHealthy
            ? Task.FromResult(HealthCheckResult.Healthy("The check indicates a healthy result."))
            : Task.FromResult(HealthCheckResult.Unhealthy("The check indicates an unhealthy result."));
    }
}
```
```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddMongoDb(Configuration["MongoConnection:ConnectionString"])
        .AddCheck<ExampleHealthCheck>("example_health_check");
    services.AddHealthChecksUI();
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseHealthChecks("/healthcheck", new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
    });
    app.UseHealthChecksUI();
}
```

![Custom health check in database](/images/2019-05-06-asp-net-core-health-checks/custom-health-dashboard.png)

As you can see, it's straightforward to add a health check with custom logic.

# Health checks by middleware
If you have a framework version smaller than 2.2, you can create a health check with the help of middleware.

```c#
public class HealthCheckMiddleware
{
    private const string Path = "/healthcheck";
    private readonly RequestDelegate _next;

    public HealthCheckMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        if (!context.Request.Path.Equals(Path, StringComparison.OrdinalIgnoreCase))
        {
            await _next(context);
        }
        else
        {
            context.Response.ContentType = "text/plain";
            context.Response.StatusCode = 200;
            context.Response.Headers.Add(HeaderNames.Connection, "close");
            await context.Response.WriteAsync("OK");
        }
    }
}
```

Add this middleware to the `Startup` class of your project.

```c#
app.UseMiddleware<HealthCheckMiddleware>();
```

# Health checks and Docker
Docker also has the support of [health checks](https://docs.docker.com/engine/reference/builder/#healthcheck).

> The HEALTHCHECK instruction tells Docker how to test a container to check that it is still working. This can detect cases such as a web server that is stuck in an infinite loop and unable to handle new connections, even though the server process is still running.

You need to add this instruction to your Dockerfile.

```dockerfile
HEALTHCHECK --interval=5m --timeout=3s CMD curl -f http://localhost:80/healthcheck || exit 1
```

Now you can check the status of your container by command `docker container ls`.

![Console with docker command](/images/2019-05-06-asp-net-core-health-checks/docker-command-console.png)

The status is `healthy`.

# Conclusion
Today we’ve looked at health checks. It’s a handy tool, especially if you are using multiple instances of your application. More information you can find in the [GitHub repository](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks) and [official docs](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-2.2), for example, health checks for different services, tools for pushing results or sending failure notifications.

*Image: Photo by Annie Spratt on Unsplash*