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

 *Disclaimer: your mileage may wary depending on data types, cache entry sizes, and other factors.*

Now onto the details. Normally, Ignite keeps cache data is serialized form in memory regions or on disk, see [Memory Architecture](https://ignite.apache.org/docs/latest/memory-architecture).
Therefore, even local read operations involve a JNI call, a copy from the memory region to the .NET memory, and a deserialization call.

[Platform Cache](https://ignite.apache.org/docs/latest/net-specific/net-platform-cache) is an additional layer of caching in the .NET memory which stores cache entries in deserialized form,
and avoids any overhead mentioned above. It is as fast as `ConcurrentDictionary`.

Some of the characteristics are:

* The cache is populated eagerly. For every local (primary, backup, or near) cache entry on the given node, an up-to-date platform cache entry exists at all times. Therefore, read operations for those entries are guaranteed to hit the platform cache - performance is predictable.
* The same instance is returned repeatedly (`ReferenceEquals(cache[1], cache[1]) == true`). Any modifications to the returned objects should be avoided: Ignite can discard them at any moment.

This feature may be not trivial to understand, but provides a huge performance improvement when used correctly. Potential scenarios are:

* 1
* 2

* TODO: List included APIs (incl ScanQuery)
* TODO: Explain like auto-synchronized ConcurrentDictionary
* TODO: Explain with ReferenceEquals, and the dangers of it 


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
