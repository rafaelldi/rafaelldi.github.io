---
title: "Proxy requests to the running .NET Docker container"
excerpt: "Sometimes I need to send some requests to a container, but its ports are not published and I don't want to restart the container. In this post, I’m going to describe a possible workaround for such a situation."
header:
  overlay_image: /assets/images/2023-01-31-proxy-docker-requests/cover.jpg
  show_overlay_excerpt: false
  caption: "Photo by [Venti Views](https://unsplash.com/@ventiviews) on [Unsplash](https://unsplash.com)"
categories: posts
author: Rival Abdrakhmanov
date: 2023-01-31
tags: ["ASP.NET Core", "Docker", "YARP", "Ports"]
sidebar:
  nav: "docker"
---

Very often a situation arises where I have to send some “debug” requests to a container, but it doesn’t have a published port. For example, I have noticed a suspicious service behavior (possibly a bug) in the test environment. To analyze the problem, I need to understand the state of the service, but it is a backend service, and it doesn’t accept any requests from outside, only from other services inside the `docker-compose` environment. Furthermore, I can’t restart the container with the published port, because I’ll lose service state. In such cases, there is a trick that has often worked for me.

# Docker container

First of all, let’s create a new plain project and add a `Dockerfile`.

```
dotnet new web --name my-app 
```

```docker
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["my-app.csproj", "./"]
RUN dotnet restore "my-app.csproj"
COPY . .
WORKDIR "/src/"
RUN dotnet build "my-app.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "my-app.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "my-app.dll"]
```

Next, build and run a container.

```
docker build -t my-app .
docker run --name my-app -d my-app
```

As you can see, I haven’t published ports from this container.

We need a separate service to proxy the requests. There are plenty of options, but I want to use [Yarp](https://github.com/microsoft/reverse-proxy).

> YARP (which stands for "Yet Another Reverse Proxy") is a project to create a reverse proxy server.

It’s fairly new and rapidly growing. Unfortunately, there is no ready-to-use docker image (see [this issue](https://github.com/microsoft/reverse-proxy/issues/247)), so we will create our own.

```
dotnet new web --name proxy
dotnet add package Yarp.ReverseProxy --version 2.0.0-rc.1.23068.3
```

I’ve added [this nuget](https://www.nuget.org/packages/Yarp.ReverseProxy) package to my project. After that, we need to modify the `Program.cs` file to enable the proxy.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.MapReverseProxy();

app.Run();
```

Configure it in the `appsettings.json` file.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ReverseProxy": {
    "Routes": {
      "my-route": {
        "ClusterId": "my-cluster",
        "Match": {
          "Path": "{**catch-all}"
        }
      }
    },
    "Clusters": {
      "my-cluster": {
        "Destinations": {
          "my-destination": {
            "Address": "http://localhost:5225/"
          }
        }
      }
    }
  }
}
```

And finally, create a separate `Dockerfile` for our new proxy service.

```docker
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["proxy.csproj", "./"]
RUN dotnet restore "proxy.csproj"
COPY . .
WORKDIR "/src/"
RUN dotnet build "proxy.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "proxy.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "proxy.dll"]
```

The next step is to decide where this proxy should send received requests. We can use `docker inspect` command to find out the address of the `my-app` service.

```
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-app
```

![Inspect container](/assets/images/2023-01-31-proxy-docker-requests/inspect-container-ip.png)

Or if you are using JetBrains Rider, there is an awesome dashboard that provides a handy way to find out different information about the containers.

![Rider Docker dashboard](/assets/images/2023-01-31-proxy-docker-requests/rider-docker-dashboard.png)

We’re almost ready. Now, build an image of the proxy container and run it (don’t forget to publish its port😉). Also, we have to set an environment variable `ReverseProxy__Clusters__my-cluster__Destinations__my-destination__Address=http://172.17.0.2/` to change the default configuration from the `appsettings.json`.

```
docker build -t my-proxy .
docker run --name my-proxy -d -p 8080:80 --env ReverseProxy__Clusters__my-cluster__Destinations__my-destination__Address=http://172.17.0.2/ my-proxy
```

Time to send some requests and check that our proxy works.

```http
GET http://localhost:8080/

HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Server: Kestrel

Hello World!
```

Yay! 🎉

# Compose service

If you’re using Docker Compose, you have to connect your container to the appropriate network. Suppose you have such a `compose.yaml` file.

```yaml
services:
  my-app:
    image: my-app
    container_name: my-app
    build:
      context: .
      dockerfile: Dockerfile
```

Let’s start it and find the network.

```
docker compose up -d
docker inspect -f '{{range $p, $conf := .NetworkSettings.Networks}}{{$p}}{{end}}' my-app
```

![Inspect container network](/assets/images/2023-01-31-proxy-docker-requests/inspect-container-network.png)

Then specify the additional parameter `--network my-app_default` in the docker run command (remember to check the IP address).

```
docker run --name my-proxy -d -p 8080:80 --network my-app_default --env ReverseProxy__Clusters__my-cluster__Destinations__my-destination__Address=http://172.19.0.2/ my-proxy
```

# Conclusion

In this post, we have discussed how to proxy a request to the container. I hope this will help you.

# References

*The source code is available [here](https://github.com/rafaelldi/containers-playground/tree/main/ports)*

- [Container networking](https://docs.docker.com/config/containers/container-networking/)
- [YARP project](https://github.com/microsoft/reverse-proxy)
- [Yarp.ReverseProxy](https://www.nuget.org/packages/Yarp.ReverseProxy)
- [Proxy is supplied as a pre-built docker image](https://github.com/microsoft/reverse-proxy/issues/247)