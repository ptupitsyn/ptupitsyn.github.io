---
layout: post
title: When ValueTask makes a big difference - DataStreamer in Ignite 3
---

`ValueTask` may seem like a micro-optimization, but it is very important on hot paths.

# DataStreamer

While working on [Data Streamer for Ignite 3](https://cwiki.apache.org/confluence/display/IGNITE/IEP-102%3A+Data+Streamer), I've added `AddWithRetryUnmapped` wrapper around synchronous `Add` method to deal with schema updates:

```csharp
await foreach (var item in data)
{    
    await AddWithRetryUnmapped(item) // was: Add(item);
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

This fairly simple change resulted in a 3x memory allocation increase in a basic streamer benchmark (100_000 items). 

```
|                    Method | ServerCount |      Mean | Allocated |
|     DataStreamer (before) |           4 |  32.64 ms |      4 MB |
|     DataStreamer (after)  |           4 |  36.41 ms |     12 MB |
```

Luckily, the fix is just as simple: replace `Task` with `ValueTask` in the method signature.

* Schema change is a rare scenario, so `AddWithRetryUnmapped` will complete synchronously most of the time.
* `Add` is being called in a loop for thousands of items, so even a small allocation adds up quickly.
