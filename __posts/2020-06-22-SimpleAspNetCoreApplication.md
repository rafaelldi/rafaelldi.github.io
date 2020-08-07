---
layout: post
title: Simple ASP.NET Core Application
date: 2020-06-22
categories: misc
---

Hello! I recently wondered what the simple ASP.NET Core application should look like. What if I'm a beginner and I want to write my first Hello world application. What is the first step? 

![Title image](https://github.com/rafaelldi/Blog/blob/master/08-SimpleAspNetCoreApplication/simple-asp-net-core-application-img.jpg)

In this article, I want to demonstrate a web server with a few lines of code and one file. If you have any experience with ASP.NET Core, this article, probably, will not be of any use to you.

# Web Server 
The main goal of this article is to show that web applications are not so challenging. If you don't know how the internet works or never heard about web servers, you can build your own in 10 minutes. Of course, it'll be the elementary solution, but that's enough for the first step.

To start with, you need to install .NET Core SDK. 

https://dotnet.microsoft.com/download/dotnet-core/3.1

Create a directory for your project, for example `Server`, and run this console command from this folder.

```
dotnet new web
```

After that, remove everything except `Server.csproj` and `Program.cs`. I've created a repository with example project on GitHub.

https://github.com/rafaelldi/SimpleAspNetCoreProject

Now, you need to modify `Program.cs` file. 

<script src="https://gist.github.com/rafaelldi/a86794ccd95991baf4541ae59bd8dad9.js"></script>

Finally, start your web server with the console command.

```
dotnet run
```

To see, that your server is working, go to `http://localhost:5000` from your browser, and you'll see this text.

![Hello world from a browser](https://github.com/rafaelldi/Blog/blob/master/08-SimpleAspNetCoreApplication/hello-world-app.png)

Congratulations, you've created a web server.

# Program.cs
If you want to go deeper, let's discuss what happens in our code. Firstly, we set up a default Generic Host with `CreateDefaultBuilder` method. This host is a container that allows us to run different tasks (implementations of IHostedService interface) with our logic. For now significant, that our web server is inherited from this interface, so we can run it too. 

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.1

With the next `ConfigureWebHostDefaults` method, we add the server to the Generic Host and configure it. We use a routing mechanism to map a path with a method. In our case, the server writes a `Hello World!` string as a response to the default `GET` request.

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.1

# Feather HTTP
Do you agree with me, that it seems a little bit complicated for a basic web server? Can we do better?

Yes, there is a fascinating library for a small servers, called Feather HTTP. With this library, we can reduce lines of code and be more specific.

https://github.com/featherhttp/framework

We need to add a new `nuget.config` file and modify `Server.csproj` to include this package in the project.

<script src="https://gist.github.com/rafaelldi/73c522f33f929693e0e84fd49d2db6d3.js"></script>

<script src="https://gist.github.com/rafaelldi/e49176eb25f96958c6fb15bea82cf32a.js"></script>

After that, we are possible to improve our `Program.cs` file. In the next code block, you'll see that we reduce the amount of code to three lines. Moreover, it's more descriptive with these changes. 

<script src="https://gist.github.com/rafaelldi/d5d912df399d8b913cdaaa4d3e82c5b0.js"></script>

# Conclusion
In this article, I showed how to create your first elementary web server. I hope that was helpful :)
