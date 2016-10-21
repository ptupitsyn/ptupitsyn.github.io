---
layout: post
title: Entity Framework As Ignite.NET Cache Store
---

Implement persistent store with Entity Framework and SQL Server.


# What Is Persistent Store

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


# Data Model

* We are going to use Entity Framework Code First approach to define the data model.
* The same classes will be used in Ignite caches.

Other approaches are possible. Ignite cache store can work in binary mode and use raw SQL to store and retrieve data from an RDBMS, for example.

There are a couple of gotchas with using EF model classes in Ignite cache without modification:

* By default, EF returns from `IDbSet` proxy objects of anonymous generated classes, which can not be serialized by Ignite. To disable this, set `DbContext.Configuration.ProxyCreationEnabled` to `false`.
* By default, `CacheConfiguration.KeepBinaryInStore` is true in Ignite, which means that ICacheStore methods receive IBinaryObject instances instead of actual objects. We want to use model classes directly, so this has to be disabled as well.
* Model classes have to be registered in BinaryConfiguration to enable Ignite serialization for them.

