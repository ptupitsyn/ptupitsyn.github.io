---
layout: post
title: Playing with C# 8.0 Async Streams in Apache Ignite
---

C# 8.0 introduces [Asynchronous Streams](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams), which combine lazy enumeration and `async`/`await`. The most obvious use case here are database queries, where every individual record is pulled asynchronously from a remote server. This applies to [Apache Ignite](https://ignite.apache.org/) too - [SQL](https://apacheignite-net.readme.io/docs/sql-queries) and [Scan](https://apacheignite-net.readme.io/docs/cache-queries#scan-queries) query APIs can be updated with async versions. This requires code modification and we can expect those things in future Ignite versions. However, there is one more query type that we can convert to async version right now - [Continuous Query](https://apacheignite-net.readme.io/docs/continuous-queries).

# Async Continuous Queries

Existing Continuous Query API is event-like: we pass an event handler class (implementing `ICacheEntryEventListener<K, V>`) and receive callbacks for all cache data modifications:

```cs
var listener = new Listener<TK, TV>();
var continuousQuery = new ContinuousQuery<TK, TV>(listener);
using (var queryHandle = cache.QueryContinuous(continuousQuery))
{
    // ...
}

// ...
private class Listener<TK, TV> : ICacheEntryEventListener<TK, TV>
{
    public void OnEvent(IEnumerable<ICacheEntryEvent<TK, TV>> events)
    {
        foreach (var cacheEntryEvent in events)
            Console.WriteLine(cacheEntryEvent.Value);
    }
}

```

As we can see, this requires quite a lot of ceremony. With async streams, we can imagine the following:

```cs
await foreach (var entry in cache.QueryContinuousAsync())
    Console.WriteLine(entry.Value);
```

The end result is the same - we handle cache update events in asynchronous, non-blocking manner; but much more concise. Let's implement this.

# QueryContinuousAsync extension method - implementation

`await foreach` construct operates on `IAsyncEnumerable<T>`, which can be produced with `yield return` - we don't need to implement the interface in our code. So an obvious start for our extension method looks like this:

```cs
public static async IAsyncEnumerable<ICacheEntry<TK, TV>> QueryContinuousAsync<TK, TV>(
    this ICache<TK, TV> cache)
{
    var queryListener = new AsyncContinuousQueryListener<TK, TV>();
    var continuousQuery = new ContinuousQuery<TK, TV>(queryListener);

    using (cache.QueryContinuous(continuousQuery))
    {
        while (???)
        {
            yield return cacheEntryEvent;
        }
    }
}

```

However, it is not clear 