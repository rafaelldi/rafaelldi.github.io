---
title: "Diagnostics tools inside Docker container"
excerpt: "This post is a collection of recipes on how to run diagnostics tools inside a docker container."
categories: posts
author: Rival Abdrakhmanov
date: 2021-11-26
tags: ["Diagnostics", ".NET", "ASP.NET Core", "Docker"]
---

![Title image](/images/2021-11-26-diagnostics-tools-inside-docker/cover_diagnostics_tools.jpg)

When you're trying to resolve a bug, you often want as much information as possible to understand the root cause of the problem. .NET provides us with valuable [diagnostics tools.](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/#net-core-diagnostic-global-tools) This post will show some scenarios of running these tools for service inside a docker container.

# Basic application

Assume that we have a .NET service, and we run it in a container. Below, I demonstrate a standard `Dockerfile` and a `docker-compose.yml` script. We will evolve them throughout the article.

```docker
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
COPY ["SimpleApp/SimpleApp.csproj", "SimpleApp/"]
RUN dotnet restore "SimpleApp/SimpleApp.csproj"
COPY . .
WORKDIR "/src/SimpleApp"
RUN dotnet build "SimpleApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SimpleApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SimpleApp.dll"]
```

```yaml
version: "3.9" 
services:
  app:
    container_name: simple
    build:
      context: .
      dockerfile: Dockerfile   
    ports:
      - "5000:80"
```

As you can see, nothing is out of the ordinary. So, the next question is how to add diagnostics tools.

# Include tools into the image

The first evident approach is to include them during the creation of our container.

```docker
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build

RUN dotnet tool install --tool-path /tools dotnet-trace
RUN dotnet tool install --tool-path /tools dotnet-counters
RUN dotnet tool install --tool-path /tools dotnet-dump
RUN dotnet tool install --tool-path /tools dotnet-gcdump

WORKDIR /src
COPY ["SimpleApp/SimpleApp.csproj", "SimpleApp/"]
RUN dotnet restore "SimpleApp/SimpleApp.csproj"
COPY . .
WORKDIR "/src/SimpleApp"
RUN dotnet build "SimpleApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SimpleApp.csproj" -c Release -o /app/publish

FROM base AS final

WORKDIR /tools
COPY --from=build /tools .

WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SimpleApp.dll"]
```

```yaml
version: "3.9" 
services:
  app:
    container_name: simple
    build:
      context: .
      dockerfile: Dockerfile   
    ports:
      - "5000:80"
    volumes:
      - /home/rafaelldi/temp:/tools/data
```

Additionally, I've added a volume to the service to facilitate access to diagnostics data. In the following snippet, I show how to collect such data. After the commands, it will be available in the `/home/rafaelldi/temp` directory on your host machine. To start collecting diagnostics data, go inside the service container and run the proper command.

```bash
docker exec -it simple bash
/tools/dotnet-counters collect -p 1 -o /tools/data/counters
/tools/dotnet-dump collect -p 1 -o /tools/data/dump
/tools/dotnet-gcdump collect -p 1 -o /tools/data/dump.gcdump
/tools/dotnet-trace collect -p 1 -o /tools/data/trace.nettrace
```

So, this is a pretty simple solution, but it's available only if you can include the tools into your release images or have separate images for your testing environment. So, if you don't want to exaggerate your release containers or build twice the amount of images, you can use a [sidecar pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar).

# Collect from a sidecar

In this solution, we don't modify the basic `Dockerfile` for our service. Instead of that, we add a `sidecar` service to the `docker-compose.yml`. This additional container includes diagnostics tools. You can find a `Dockerfile` for it bellow. Also, we have to create a `tmp` volume to "connect" both containers through it.

```yaml
version: "3.9"
services:
  app:
    container_name: simple
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:80"
    volumes:
      - tmp:/tmp
      - /home/rafaelldi/temp:/tools/data

  sidecar:
    container_name: diagnostics
    build:
      context: .
      dockerfile: SidecarDockerfile
    volumes:
      - tmp:/tmp
      - /home/rafaelldi/temp:/tools/data
    tty: true

volumes:
  tmp:
```

```docker
FROM mcr.microsoft.com/dotnet/sdk:5.0 as build

RUN dotnet tool install --tool-path /tools dotnet-trace
RUN dotnet tool install --tool-path /tools dotnet-counters
RUN dotnet tool install --tool-path /tools dotnet-dump
RUN dotnet tool install --tool-path /tools dotnet-gcdump

FROM mcr.microsoft.com/dotnet/runtime:5.0 AS final
COPY --from=build /tools /tools

WORKDIR /tools
ENTRYPOINT ["bash"]
```

To run our diagnostics tools, we need to step inside the `diagnostics` container and execute a command.

```bash
docker exec -it diagnostics bash
/tools/dotnet-counters collect -p 1 -o /tools/data/counters
/tools/dotnet-dump collect -p 1 -o /tools/data/dump
/tools/dotnet-gcdump collect -p 1 -o /tools/data/dump.gcdump
/tools/dotnet-trace collect -p 1 -o /tools/data/trace.nettrace
```

This approach has one disadvantage. You have to keep the `sidecar` container inside your `docker-compose.yml` file. That doesn't sound like a neat solution for a production environment. If you don't want to do that, it's possible to attach from a separate container.

Notice, that we have a similar to the basic docker-compose script, but we still need an additional `tmp` volume.

```yaml
version: "3.9"
services:
  app:
    container_name: simple
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:80"
    volumes:
      - tmp:/tmp
      - /home/rafaelldi/temp:/tools/data

volumes:
  tmp:
    name: tmp
```

```docker
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build

RUN dotnet tool install --tool-path /tools dotnet-trace

FROM mcr.microsoft.com/dotnet/runtime:5.0 AS final
COPY --from=build /tools /tools

WORKDIR /tools
ENTRYPOINT ["/tools/dotnet-trace", "collect", "-p", "1", "-o", "/tools/data/trace.nettrace"]
```

New `Diagnostics` image has only `dotnet-trace` tool installed. You can create pretty same images for other tools and save them, for example, to your private docker registry.

When you execute the `docker run` command, the container starts, "connects" to your service and starts collecting the diagnostics data.

```bash
docker build -t diagnostics -f DiagnosticsDockerfile .
docker run --rm --pid=container:simple -v tmp:/tmp -v /home/rafaelldi/temp:/tools/data -it diagnostics
```

There is another solution if you don't want to keep even a `tmp` volume inside your production `docker-compose.yml`.

# Download tools into the container

You can download the tools to the service container, collect the data and remove them. For example, you've discovered a strange behaviour of the production service and want to get some traces or dumps to understand what's going on. I've put self-explanatory instructions below.

```bash
1. Get into the container.
docker exec -it simple bash

2. Preliminary.
mkdir /tools
cd /tools
mkdir ./data
apt-get update
apt-get install curl

3. Download the tools.
curl -JLO https://aka.ms/dotnet-counters/linux-x64 && chmod +x ./dotnet-counters
curl -JLO https://aka.ms/dotnet-dump/linux-x64 && chmod +x ./dotnet-dump
curl -JLO https://aka.ms/dotnet-gcdump/linux-x64 && chmod +x ./dotnet-gcdump
curl -JLO https://aka.ms/dotnet-trace/linux-x64 && chmod +x ./dotnet-trace

4. Collect a data.
/tools/dotnet-counters collect -p 1 -o /tools/data/counters
/tools/dotnet-dump collect -p 1 -o /tools/data/dump
/tools/dotnet-gcdump collect -p 1 -o /tools/data/dump.gcdump
/tools/dotnet-trace collect -p 1 -o /tools/data/trace.nettrace

5. Copy the data to the host machine.
docker cp simple:/tools/data/trace.nettrace /home/rafaelldi/temp/
sudo chmod 755 /home/rafaelldi/temp/trace.nettrace
```

# JetBrains profilers

Lastly, I'm going to show some commands for using [JetBrains profilers](https://www.jetbrains.com/dotnet/). These are more sophisticated and therefore paid tools.

There is a `dotTrace` global tool, so it's possible to use it similarly as a `dotnet-trace`.

```bash
dotnet tool install --global JetBrains.dotTrace.GlobalTools
```

Or you can download `dotTrace` and `dotMemory` inside your service container.

```bash
1. Get into the container.
docker exec -it simple bash

2. Preliminary.
mkdir /tools
cd /tools
mkdir ./data
mkdir ./dotTrace
mkdir ./dotMemory
apt-get update
apt-get install curl

3. Download the tools.
curl -L -o dotTrace.tar.gz https://download.jetbrains.com/resharper/dotUltimate.2021.2.2/JetBrains.dotTrace.CommandLineTools.linux-x64.2021.2.2.tar.gz && tar -xf dotTrace.tar.gz -C /tools/dotTrace
curl -L -o dotMemory.tar.gz https://download.jetbrains.com/resharper/dotUltimate.2021.2.2/JetBrains.dotMemory.Console.linux-x64.2021.2.2.tar.gz && tar -xf dotMemory.tar.gz -C /tools/dotMemory

4. Collect a data.
/tools/dotTrace/dottrace attach 1 --save-to=/tools/data/snapshot.dtp --timeout=10s
/tools/dotMemory/dotmemory get-snapshot 1 --save-to-dir=/tools/data

5. Copy the data to the host machine.
docker cp simple:/tools/data/snapshot.dtp /home/rafaelldi/temp/
```

# Conclusion

Sometimes, you need some traces or dumps from your services in a testing or production environment to understand the bug or a particular problem. In this post, I've shown some ways to do that with .NET diagnostics tools.

# References

- [What diagnostic tools are available in .NET Core?](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/)
- [Collect diagnostics in containers](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/diagnostics-in-containers)

*Image: Photo by Hunter Haley on Unsplash*

