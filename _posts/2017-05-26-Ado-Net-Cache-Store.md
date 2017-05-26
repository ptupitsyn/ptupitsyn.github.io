---
layout: post
title: ADO.NET As Ignite.NET Cache Store
---

Implementing efficient Ignite.NET persistent store with ADO.NET and SQL Server: continue the story from [Entity Framework Cache Store](https://ptupitsyn.github.io/Entity-Framework-Cache-Store/).


# ADO.NET vs Entity Framework

Previous article, [Entity Framework Cache Store](https://ptupitsyn.github.io/Entity-Framework-Cache-Store/), describes a way to persist Ignite in-memory data in SQL server using Entity Framework. The code is nice and elegant, but not efficient (as mentioned on [reddit](https://www.reddit.com/r/programming/comments/593ep1/entity_framework_as_ignitenet_cache_store/d95mnt6/)), because converting object operations to SQL queries introduces overhead.

Today we are going to cut out all the middlemen:

* Work with SQL directly to read and write data
* Use binary mode in Ignite to avoid serialization costs

# Ignite 2.0 Generic Cache Store

Cache store interface has been reworked in Ignite.NET 2.0 to operate on generic arguments. This reduces casting and boxing, making code nicer and faster:

```cs
// Ignite 1.x
class MyStore : ICacheStore
{
    public object Load(object key) => db.Find((int) key);
}

// Ignite 2.x
class MyStore : ICacheStore<int, string>
{
    public string Load(int key) => db.Find(key);
}

```

# Ignite Binary Mode

By default, Ignite works with user-defined objects and types, serializing / deserializing them as needed. While this serialization is very [efficient](https://ptupitsyn.github.io/Ignite-Serialization-Performance/), it is still not free.

To squeeze every bit of performance there is a [binary mode](https://apacheignite-net.readme.io/docs/binary-mode) where we work with objects in serialized form, retrieving and modifying individual fields.

We are going to use this binary mode both on cache side and cache store side.

# Code

The full source code is on [github.com/ptupitsyn/ignite-net-examples](https://github.com/ptupitsyn/ignite-net-examples), under AdoNetCacheStore folder.

The project is self-sufficient, you can download the sources and run it without setting anything up.
It uses SQL Server Compact (via NuGet) and creates a database in the bin folder when needed.

# Data Model

