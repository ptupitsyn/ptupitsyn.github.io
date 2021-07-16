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
            result = _cache[count - 1];

            return true;
        }
    }
}
```

If multiple consumers compete for an item and call `Replace` in parallel, only one call will succeed, and only one client will process an item with the given ID.
Other clients will continue looping and trying to get the next item, if any.

# Blocking Take and Continuous Query

# Closing

* Add cache configuration
* Implement interfaces: `IEnumerable`, `ICollection`, `IProducerConsumerCollection`
* `TakeAsync`
