---
title: "GraphQL with ASP.NET Core"
excerpt: "I’m continuing my services communication series with GraphQL."
header:
  og_image: /assets/images/2020-02-23-graphql-with-asp-net-core/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2020-02-23
tags: ["Connection", "ASP.NET Core", "GraphQL"]
---

![Title image](/assets/images/2020-02-23-graphql-with-asp-net-core/cover.jpg)

I’m continuing my services communication series with GraphQL.

I’ve created a [repository](https://github.com/rafaelldi/GraphQLWithAspNetCore) with an example project.

# GraphQL
[GraphQL](https://graphql.org/) is a query language for API. You provide only one endpoint and use it for all requests. A client describes with special syntax its needs and sends to your endpoint.

GraphQL is language-agnostic, and there are libraries for [all popular frameworks](https://graphql.org/code/).

# Server
First of all, we’ll create a server. GrapQL isn’t supported by ASP.NET Core official, so we need to instal some nuget packages. I am going to show you how to work with [GraphQL.NET library](https://github.com/graphql-dotnet), and then we’ll discuss another option.

Create a new project with a command.

```
dotnet new web
```

After that, add these packages:

```
GraphQL 
GraphQL-Parser 
GraphQL.Server.Transports.AspNetCore 
GraphQL.Server.Transports.AspNetCore.SystemTextJson
```

To use GraphQL, we need to describe all types, queries, mutations and schema in our project. Sadly, it’s too wordy.

At first, add our type.

```csharp
public class ArticleType : ObjectGraphType<Article>
{
    public ArticleType()
    {
        Field(x => x.Title);
        Field(x => x.Author);
        Field(x => x.Content);
    }
}
```

If you want to request your objects, you need to add a query.

```csharp
public class BlogQuery : ObjectGraphType
{
    public BlogQuery(ArticleService articleService)
    {
        Field<ListGraphType<ArticleType>>("list", resolve: context => articleService.Articles);
    }
}
```

If you want to add or update some objects, you need to create a mutation and describe an input type.

```csharp
public class BlogMutation : ObjectGraphType
{
    public BlogMutation(ArticleService articleService)
    {
        Field<ArticleType>(
            "create",
            arguments: new QueryArguments
            {
                new QueryArgument<NonNullGraphType<ArticleInputType>> { Name = "article" }
            },
            resolve: context =>
            {
                var article = context.GetArgument<Article>("article");
                articleService.Articles.Add(article);
                return article;
            });
    }
}
```

```csharp
public class ArticleInputType : InputObjectGraphType<Article>
{
    public ArticleInputType()
    {
        Field(x => x.Title);
        Field(x => x.Author);
        Field(x => x.Content);
    }
}
```

Eventually, put it all together in a schema.

```csharp
public class BlogSchema : Schema
{
    public BlogSchema(ArticleService articleService)
    {
        Query = new BlogQuery(articleService);
        Mutation = new BlogMutation(articleService);
    }
}
```

Ok, it’s time to deal with `Startup` class. Because there is only one endpoint, we don’t need any controllers. Every request will be handled by a single middleware. As always, I use an `ArticleService` to imitate a database.

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton<ArticleService>();
        services.AddScoped<BlogSchema>();

        services.AddGraphQL()
            .AddGraphTypes(ServiceLifetime.Scoped)
            .AddSystemTextJson();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseGraphQL<BlogSchema>();
    }
}
```

Now, our simple server is complete. I think that there wasn’t anything challenging for you.

How can we test our solution? The great news that [Postman](https://www.postman.com/) supports GrapQL. It’s very popular tool for REST API, so you won’t spend a lot of time to study it.

![Mutation in graphql](/assets/images/2020-02-23-graphql-with-asp-net-core/mutation-graphql.png)

![Query in graphql](/assets/images/2020-02-23-graphql-with-asp-net-core/query-graphql.png)

Another way is to use specific tools for GraphQL, for example, GraphiQL. For this purpose, we need to install a new package `GraphQL.Server.Ui.GraphiQL` and add the following line to the `Startup`.

```csharp
app.UseGraphiQLServer(new GraphiQLOptions());
```

This adds a UI for requesting your API. Go to `http://localhost:5000/graphiql` and try it.

![Query in GraphiQL](/assets/images/2020-02-23-graphql-with-asp-net-core/query-graphiql.png)

GraphiQL supports code completion and contains all queries and mutations from your project, so it’s a convenient tool.

![Documentation explorer](/assets/images/2020-02-23-graphql-with-asp-net-core/documentation-explorer.png)

As you see, I use these expressions for calling our API. GraphQL has its [query syntax](https://graphql.org/learn/queries/). I’ll show you some other examples later.

```graphql
mutation($article: ArticleInputType!) {
 create(article: $article) {
  title  
  author  
  content
 }
}
```

```graphql
query {
 list {
  title
  author  
  content
 }
}
```

# Client
Now, let’s create a GraphQL client. I use [GraphQL.Client library](https://github.com/graphql-dotnet/graphql-client) and the same worker template as in the previous posts. You have to put ordinary HttpClient in the GraphQL wrapper with .AsGraphQLClient() extension method. After that, you can use this client for sending requests. Here is the code.

```csharp
public class Worker : BackgroundService
{
    private readonly IHttpClientFactory _factory;
    private const string CreateCommand = "create";
    private const string GetListCommand = "get-list";

    public Worker(IHttpClientFactory factory)
    {
        _factory = factory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("Type a command.");
            var command = Console.ReadLine();
            if (command != CreateCommand && command != GetListCommand)
            {
                Console.WriteLine("Wrong command. Try again.");
                continue;
            }
            
            using var client = _factory.CreateClient("localhost");
            using var graphqlClient = client.AsGraphQLClient("http://localhost:5000/graphql");

            if (command is GetListCommand)
            {
                var getRequest = new GraphQLHttpRequest
                {
                    Query = @"query
{
    list {
        title
        author  
        content
    }
}"
                };
                var response = await graphqlClient.SendQueryAsync<GraphQLRespone>(getRequest, stoppingToken);
                Console.WriteLine($"We have {response.Data.List.Count} articles");
                foreach (var article in response.Data.List)
                {
                    Console.WriteLine($"Article '{article.Title}' by {article.Author}");
                }
            } else if (command is CreateCommand)
            {
                Console.WriteLine("Type a title");
                var title = Console.ReadLine();
                Console.WriteLine("Type an author");
                var author = Console.ReadLine();
                Console.WriteLine("Type a content");
                var content = Console.ReadLine();
                
                var article = new Article {Title = title, Author = author, Content = content};
                var createRequest = new GraphQLHttpRequest()
                {
                    Query = @"mutation($article: ArticleInputType!) 
{
    create(article: $article) {
        title  
        author  
        content
    }
}",
                    Variables = new {
                        article
                    }
                };
                await graphqlClient.SendMutationAsync<Article>(createRequest, stoppingToken);
            }
        }
    }
}
```

# Requests
Until now, everything was similar to REST, and you may ask what the benefits are. The power of GraphQL lies in its query syntax. A client specifies which fields he wants to obtain. For example, we need only the titles of our articles.

```graphql
query{
  list{
    title
  }
}
```

![Request in GraphiQL](/assets/images/2020-02-23-graphql-with-asp-net-core/request-graphiql.png)

We have to modify a little query, and the server responds to us with the required information, without any changes in the server.

Further, let’s consider filtration. Create a second query with a title parameter and choose appropriate values with LINQ expression.

```csharp
Field<ListGraphType<ArticleType>>(
    "filtered_list", 
    arguments: new QueryArguments
    {
        new  QueryArgument<StringGraphType> { Name = "title"}    
    },
    resolve: context =>
    {
        var titleFilter = context.GetArgument<string>("title");
        return articleService.Articles.Where(a => a.Title == titleFilter);
    });
```

![Filtered request in GraphiQL](/assets/images/2020-02-23-graphql-with-asp-net-core/filtered-request-graphiql.png)

# Hot Chocolate
There is another library to support GraphQL in your project called [Hot Chocolate](https://hotchocolate.io/). In my opinion, it has more complete documentation, handy tooling and better support. If you are going to deal with GraphQL, I advise you to take a look at this library.

# Conclusion
Today we briefly run through the GraphQL in ASP.NET Core application.

To summarize, I think that GraphQL may be a powerful tool to manage plenty of non-trivial queries. For example, you may have an elaborate graph of objects and different consumers with variable needs. In that way, it’s more sensible to give an ability to your clients to describe their requests rather than create a version of your API for each consumer.

In the other hand, GraphQL has drawbacks. You need to learn its query syntax; there is no caching; you can’t adjust your SQL queries for better performance. You have to think about security differently, for instance, the client can send you dozens of heavy requests and produce a DDoS attack. So, plenty of things demand your attention.

# References
* [GraphQL](https://graphql.org/);
* [GraphQL.NET](https://graphql-dotnet.github.io/);
* [Hot Chocolate](https://hotchocolate.io/).

*Image: Photo by Alfons Morales on Unsplash*


