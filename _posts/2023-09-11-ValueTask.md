---
layout: post
title: When ValueTask makes a big difference - DataStreamer in Ignite 3
---

`ValueTask` may seem like a micro-optimization, but it is very important on hot paths, and easy to miss. Last week I've learned this the hard way.

# Async on Hot Path

While working on [Data Streamer](https://cwiki.apache.org/confluence/display/IGNITE/IEP-102%3A+Data+Streamer) for [Apache Ignite 3](https://ignite.apache.org/), 
I've added `AddWithRetryUnmapped` wrapper around synchronous `Add` method to [deal with schema updates](https://github.com/apache/ignite-3/commit/5ac19cfb9528ec2a72edd1e5a1ff3d03f24b4537):

```csharp
await foreach (var item in data)
{    
    await AddWithRetryUnmapped(item); // was: Add(item);
}

async Task<(Batch<T> Batch, string Partition)> AddWithRetryUnmapped(T item)
{
    try
    {
        return Add(item);
    }
    catch (Exception e) when (e.CausedByUnmappedColumns())
    {
        schema = await schemaProvider(Table.SchemaVersionForceLatest).ConfigureAwait(false);
        return Add(item);
    }
}
```

This fairly simple change resulted in a 3x memory allocation increase in a [basic streamer benchmark](https://github.com/apache/ignite-3/blob/5ac19cfb9528ec2a72edd1e5a1ff3d03f24b4537/modules/platforms/dotnet/Apache.Ignite.Benchmarks/Table/DataStreamerBenchmark.cs) (100_000 items). 

```
|                    Method | ServerCount |      Mean | Allocated |
|     DataStreamer (before) |           4 |  32.64 ms |      4 MB |
|     DataStreamer (after)  |           4 |  36.41 ms |     12 MB |
```

Luckily, the fix is just as simple: replacing `Task` with `ValueTask` in the method signature brings the performance back to normal.

* Schema change is a rare scenario, so `AddWithRetryUnmapped` will complete synchronously most of the time
* `Add` is being called in a loop for thousands of items, so even a small allocation adds up quickly

`ValueTask` has almost no overhead when the method completes synchronously - it is just a small `struct`, 
so the addition of `AddWithRetryUnmapped` does not affect performance when there are no schema changes.


# Conclusion

* Benchmark every change to catch performance regressions
* Use `ValueTask` when there is any chance that the method will complete synchronously

# Links

* [Understanding the Whys, Whats, and Whens of ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
* [The performance characteristics of async methods in C#](https://devblogs.microsoft.com/premier-developer/the-performance-characteristics-of-async-methods/)
