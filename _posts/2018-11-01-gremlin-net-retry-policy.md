---
layout: post
title: "Implementing retry policies for Gremlin.NET"
date: 2018-11-01 19:30:00 +0300
categories: programming
tags: gremlin.net, retry, policies, graph, cosmosdb, azure
---

At the time of writing this article, [Gremlin.NET](https://github.com/apache/tinkerpop/tree/master/gremlin-dotnet) client library does not support retrying after transient faults. This is a big issue when client communicate with graph services which have to provide  some kind of SLA agrrements.

For instance, Azure Cosmos DB to fit SLA can break some requests with 429 status code, which means that you need to wait before you can do another attempt. This is an example of [throttling](https://docs.microsoft.com/en-us/azure/architecture/patterns/throttling).

As there is no retrying mechanism in the Gremlin.NET client, we have to implement it by our own. When cosmos graph database starts to consume more RU (request units) than configured, the service starts sending responces with 429 status code and `retry-after` header, which tells us how much time we have to wait before send another request.

A bad news are that up to `v3.4.0-rc2` client does not provide any intormation about headers. That means we cannot use `retry-after` header. So in that case we can implement only exponential retry, which may be much slower.

Here is what `ResponseExceptions` contains starting at `v3.4.0-rc2` version:

![Learning Path]({{ "/assets/gremlin-retry/ResponseExceptionContent.png" | absolute_url }})

Let's write some code for both cases (exponential retry and `retry-after` header)

For instance, we have the following query:

```cs
    internal interface IQuery<T>
    {
        Task<T> Execute();
    }

    internal class GraphQuery : IQuery<IEnumerable<Vertex>>
    {
        private readonly GremlinClient _graph;

        public GraphQuery(GremlinClient graph)
        {
            _graph = graph;
        }

        public async Task<IEnumerable<Vertex>> Execute()
        {
            ResultSet<Vertex> result = await _graph
                .SubmitAsync<Vertex>("g.V().outE().hasLabel('EXCLUDES').inV()");

            return result;
        }
    }
```

Query decorator for exponential retry may look like the following:

```cs
    internal class GraphQueryWithExponentialBackOff<T> : IQuery<T>
    {
        private readonly Policy _retryPolicy;
        private readonly IQuery<T> _origin;
        
        public GraphQueryWithExponentialBackOff(
            RetryOptions options,
            IQuery<T> origin)
        {
            _origin = origin;
            _retryPolicy = Policy
                .Handle<ResponseException>()
                .WaitAndRetryAsync(
                    options.RetryCount,
                    attempt => Enumerable
                        .Repeat(options.WaitTime, options.RetryCount)
                        .Aggregate(TimeSpan.Zero, (result, current) => result.Add(current)));
        }

        public Task<T> Execute()
        {
            return _retryPolicy.ExecuteAsync(() => _origin.Execute());
        }
    }
```

Query decorator which uses `retry-after` header may be implemented is a such way:

```cs
    internal class GraphQueryWithServerBackOff<T> : IQuery<T>
    {
        private const int RetryAfterStatusCode = 429;
        
        private readonly IQuery<T> _origin;
        private readonly RetryOptions _options;
        
        public GraphQueryWithServerBackOff(
            RetryOptions options,
            IQuery<T> origin)
        {
            _options = options;
            _origin = origin;
        }

        public Task<T> Execute()
        {
            return Policy
                .Handle<ResponseException>(exception => exception.CosmosDbStatusCode() == RetryAfterStatusCode)
                .WaitAndRetryAsync(
                    _options.RetryCount,
                    (attempt, exception, _) => ((ResponseException) exception).CosmosDbRetryAfter(),
                    (exception, waitTime, attempt, _) => Task.CompletedTask)
                .ExecuteAsync(() => _origin.Execute());
        }
    }
```

For better code redability the following extension methods can be created:

```cs
    public static class ResponseExceptionExtensions
    {
        public static int CosmosDbStatusCode(this ResponseException source)
        {
            if (!source.StatusAttributes.TryGetValue("x-ms-status-code", out var code))
            {
                throw new InvalidOperationException("Header 'x-ms-status-code' is not presented.");
            }

            return Int32.Parse(code.ToString());
        }
        
        public static TimeSpan CosmosDbRetryAfter(this ResponseException source)
        {
            if (!source.StatusAttributes.TryGetValue("x-ms-retry-after-ms", out var time))
            {
                throw new InvalidOperationException("Header 'x-ms-status-code' is not presented.");
            }

            return TimeSpan.Parse(time.ToString());
        }
    }
```
Happy coding!