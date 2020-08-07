---
layout: post
title: "Simple Giraffe Application"
categories: misc
---

Last time we talked about the simplest ASP.NET Core application in C#. Today we'll create the same project in a functional way.

![Title image](https://raw.githubusercontent.com/rafaelldi/rafaelldi.github.io/master/images/simple-giraffe-application-img.jpg)

Recently, I've been interested in functional programming because it's a totally different approach with intriguing challenges. Also, I think you noticed that C# sometimes looks functional, so I was wondered how much differences between two elementary web servers would be. Hence, I decided to make a post with a "Hello world" example in F#.

# Creation from a template

First of all, create an application from the ASP.NET template.

```
dotnet new web -lang F#
```

We need only `Server.fsproj` and `Program.fs`, other generated files you can safely delete.

# Giraffe

There is a library for creating web applications in a functional approach. We'll use it for our server.

> A native functional ASP.NET Core web framework for F# developers. 

https://github.com/giraffe-fsharp/Giraffe

To add this package to the project let's modify `.fsproj` file. It's similar to the ordinary C# project files.

<script src="https://gist.github.com/rafaelldi/ce4b1c048d67c7ea5436a259aaba0884.js"></script>

Now, alter a code from the `Program.fs`. 

<script src="https://gist.github.com/rafaelldi/9e9a7ca61a7c3f3d89331b0f3153e64c.js"></script>

In the beginning, we describe our routes. Here we have only one endpoint. The rest of this file will be familiar to you, compare it to the previous example.

That's all! Finally, we have a functional web server. You can run it and test with a request from a browser.

# Conclusion

In this post, I described a straightforward example of a functional web server. With .NET, it's effortlessy to switch to F# and spend some time learning functional style. Maybe in the future, I'll publish some new posts about F#.

https://github.com/rafaelldi/SimpleGiraffeApplication
