---
layout: post
title: Apache Ignite Thin Client Distributed Blocking Queue
---

Apache Ignite "thick" API provides a number [Distributed Data Structures](https://ignite.apache.org/features/datastructures.html), such as Queues and Atomics. Those APIs are not yet available in thin clients, but we can easily implement them on top of the Cache API.     


# Distributed Blocking Queue

Let's say we want to implement a cluster-wide distributed producer-consumer scenario, where any client can enqueue items,
and one or more clients process them in a loop:

```cs
while (true)
    Process(queue.Take()); // Blocks if there are no items
```

Requirements:
* Any number of producers and consumers in the cluster.
* Every item is processed once and only once.
* `Take` blocks if there are no items available, and unblocks immediately when item arrives.


# Atomic Cache Operations and Non-Blocking Synchronization

Ignite guarantees cluster-wide atomicity for all cache operations, such as `Cache.Replace(key, old, new)`.

Using this fact we can implement safe concurrent `Enqueue` and `Dequeue` operations using an ID counter:

```cs
public bool TryDequeue(out T result)
{
    while (true)
    {
        var count = _cacheCounter[CounterId];

        if (count == 0)
        {
            result = default;
            return false;
        }

        if (_cacheCounter.Replace(key: CounterId, oldVal: count, newVal: count - 1))
        {
            result = _cache.GetAndRemove(count - 1).Value;

            return true;
        }
    }
}
```

If multiple consumers compete for an item and call `Replace` in parallel, only one call will succeed, and only one client will process an item with the given ID.
Other clients will continue looping and trying to get the next item, if any.


# Blocking Take and Continuous Query

`TryDequeue` above is non-blocking and will return `false` if the queue is empty, which is useful in some situations.
But for our producer-consumer scenario we need a `Take` method which will block until an item is available.
We could use a polling approach with repeated `TryTake` calls in a loop, probably with some delay in between, but this is inefficient - we'll burn CPU cycles and waste network bandwidth for client requests.

Instead, we can use `ContinuousQuery` to get a notification when new items are added to the cache:

```cs
public T Take()
{
    if (TryDequeue(out var result))
        return result;

    lock (_querySyncRoot)
    {
        using var query = _cache.QueryContinuous(new ContinuousQueryClient<int, T>(this));

        while (true)
        {
            if (TryDequeue(out result))
                return result;

            Monitor.Wait(_querySyncRoot);
        }
    }
}

void ICacheEntryEventListener<int, T>.OnEvent(IEnumerable<ICacheEntryEvent<int, T>> evts)
{
    lock (_querySyncRoot)
        Monitor.Pulse(_querySyncRoot);
}
```

# Closing

Minimal implementation with tests is available on GitHub: [github.com/ptupitsyn/ignite-net-examples/tree/master/ThinClientQueue](https://github.com/ptupitsyn/ignite-net-examples/tree/master/ThinClientQueue).

To make this solution complete, I would add the following features:

* Optionally pass cache configuration to the constructor: we can tweak backups, or even make our queue persistent to disk. 
* Implement interfaces: `IEnumerable`, `ICollection`, `IProducerConsumerCollection`
* Implement `TakeAsync` to avoid blocking consumer threads while preserving the same loop logic: `while (true) Process(awaite queue.TakeAsync());`
