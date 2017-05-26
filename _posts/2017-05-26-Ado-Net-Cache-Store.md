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

Cache store interface has been reworked in Ignite.NET 2.0: TODO

# Code

The full source code is on [github.com/ptupitsyn/ignite-net-examples](https://github.com/ptupitsyn/ignite-net-examples), under AdoNetCacheStore folder.

The project is self-sufficient, you can download the sources and run it without setting anything up.
It uses SQL Server Compact (via NuGet) and creates a database in the bin folder when needed.


# Data Model

