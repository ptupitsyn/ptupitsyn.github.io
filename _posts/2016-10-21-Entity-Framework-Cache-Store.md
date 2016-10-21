---
layout: post
title: Entity Framework As Ignite.NET Cache Store
---

Implement persistent store with Entity Framework and SQL Server.

# What is Persistent Store

Apache Ignite stores cached data in RAM.
Fault tolerance can be achieved by [configuring caches in a certain way](https://apacheignite.readme.io/docs/cache-modes).

However, there are use cases where disk-based storage may be needed:

* Store all data on disk so it survives cluster restarts.
* Use Ignite as a cache in front of disk-based RDBMS to improve performance.

Cache store is represented by `ICacheStore` interface,
which is called by Ignite when certain `ICache` methods are called [(more details)](https://apacheignite-net.readme.io/docs/persistent-store).

In this post we are going to create `ICacheStore` implementation that delegates all operations to the EntityFramework.

# Code

The full source code is on [github.com/ptupitsyn/ignite-net-examples](https://github.com/ptupitsyn/ignite-net-examples), under EFCacheStore folder.

The project is self-sufficient, you can download the sources and run it without setting anything up.
It uses SQL Server Compact (via NuGet) and creates a database in the bin folder when needed.



