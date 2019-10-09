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
        while (true)
        {
            cacheEntryEvent = await ???;
            yield return cacheEntryEvent;
        }
    }
}

```

The problem is - what do we await here? Ignite invokes our `AsyncContinuousQueryListener` on every cache update, and we need to feed those updates to the loop above. This is a classic producer-consumer scenario, which is typically done with `BlockingCollection<T>` or [other blocking thread synchronization primitives](http://www.albahari.com/threading/part4.aspx#_Wait_Pulse_Producer_Consumer_Queue). The key word here is `blocking`, and we don't want to block our threads, that's the whole idea of `async`.

We need something like `BlockingCollection.TakeAsync`, or `AutoResetEvent.WaitOneAsync`, neither of which exist, unfortunately. The problem is discussed in detail on StackOverflow: [Awaitable AutoResetEvent](https://stackoverflow.com/questions/32654509/awaitable-autoresetevent). It turns out, `SemaphoreSlim` has `WaitAsync` method, which fits the purpose, and the result is:

```cs
public static class IgniteAsyncStreamExtensions
{
    public static async IAsyncEnumerable<ICacheEntry<TK, TV>> QueryContinuousAsync<TK, TV>(this ICache<TK, TV> cache)
    {
        var queryListener = new AsyncContinuousQueryListener<TK, TV>();
        var continuousQuery = new ContinuousQuery<TK, TV>(queryListener);

        using (cache.QueryContinuous(continuousQuery))
        {
            while (true)
            {
                while (queryListener.Events.TryDequeue(out var entryEvent))
                    yield return entryEvent;

                await queryListener.HasData.WaitAsync();
            }
        }
    }

    private class AsyncContinuousQueryListener<TK, TV> : ICacheEntryEventListener<TK, TV>
    {
        public readonly SemaphoreSlim HasData = new SemaphoreSlim(0, 1);

        public readonly ConcurrentQueue<ICacheEntryEvent<TK, TV>> Events = new ConcurrentQueue<ICacheEntryEvent<TK, TV>>();

        public void OnEvent(IEnumerable<ICacheEntryEvent<TK, TV>> events)
        {
            foreach (var entryEvent in events)
                Events.Enqueue(entryEvent);

            HasData.Release();
        }
    }
}

```

This works! Important thing to note is that Ignite `QueryContinuous` method returns `IDisposable`, which should be disposed in order to stop the continuous query. But our `QueryContinuousAsync` implementation contains an infinite loop within `using` statement - there is no `break`! Does it mean that continuous query will never stop filling the `Events` queue, wasting memory? Not at all! As we know, C# compiler transforms methods with `yield return` into 