---
layout: post
title: What's new in Apache Ignite.NET 2.8
---

[Apache Ignite](https://ignite.apache.org/) 2.9 [has been released](http://apache-ignite-users.70518.x6.nabble.com/ANNOUNCE-Apache-Ignite-2-9-0-Released-td34311.html) a few days ago.
Let's have a look at .NET-specific features and improvements.  


# Platform Cache: It's All About Performance

```
|               Method |        Mean | Ratio | Allocated |
|--------------------- |------------:|------:|----------:|
|             CacheGet | 3,015.56 ns | 69.98 |    4176 B |
| CacheGetWithPlatform |    43.09 ns |  1.00 |      32 B |
```

**70 times faster**, not bad? The code is on GitHub: [PlatformCacheBenchmark.cs](https://github.com/ptupitsyn/IgniteNetBenchmarks/blob/bab8535a4a22e7e863a9929f590bbb9a80140fcf/PlatformCacheBenchmark.cs).

Now onto the details: Ignite keeps cache data in serialized form in memory regions or on disk (see [Memory Architecture](https://ignite.apache.org/docs/latest/memory-architecture)).
Therefore, even local read operations involve a JNI call, a copy from the memory region to the .NET memory, and a deserialization call.

[Platform Cache](https://ignite.apache.org/docs/latest/net-specific/net-platform-cache) is an additional layer of caching in the .NET memory which stores cache entries in deserialized form,
and avoids any overhead mentioned above. It is as fast as `ConcurrentDictionary`.

Naturally, there are tradeoffs: memory usage is increased, and cache write performance is affected. This feature is best suited for read-only or rarely changing data. 
Platform Cache can be used on client and server nodes, see [documentation](https://ignite.apache.org/docs/latest/net-specific/net-platform-cache) for more details.


### Scan Query with Platform Cache

TODO: Benchmark


# Call .NET Services From Java: Full Circle of Services

Effectively, .NET services are now first-class citizens and can be called from anywhere: thick and thin clients, Java or .NET.
And, potentially, from other thin clients, like Python or Rust - the protocol supports that (TODO: Links to other clients).

TODO: This works from Java Thin as well!



# IgniteLock: Cache.Lock Replacement

TODO: Cache.Lock should be marked Obsolete 


# Thin Client Automatic Server Discovery

# Thin Client Compute

TODO: Java and .NET services!

# Other Improvements

* SqlFieldsQuery as ContinuousQuery.InitialQuery 
* FieldsQueryCursor metadata
* AffinityCall/AffinityRun with partition - mention this in Platform Cache section instead?
* Thin client cluster APIs - mention in server discovery?


# Wrap-up

TODO:
* Full release notes: https://ignite.apache.org/releases/2.9.0/release_notes.html
* New docs: https://ignite.apache.org/docs/latest/
