---
layout: post
title: What's new in Apache Ignite.NET 2.1
---

Apache Ignite 2.1 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-1-a) today, introducing Ignite Persistent Store!
Another huge step forward for the project: it becomes a real database with unique feature set, just look at the comparison table on [ignite.apache.org frontpage](https://ignite.apache.org/). Let's have a look at new features from .NET standpoint.

![ignite logo](../images/ignite_logo.png)


# Ignite Persistent Store

Ignite has been an in-memory system from the start: RAM is fast, disk is slow, we all know that.
However, in most use cases we still want to persist data in some non-volatile storage, in case of full cluster restart, data center failures, and the like.
Most common solution is an RDBMS serving as [cache store](https://apacheignite-net.readme.io/docs/persistent-store), which has a lot of downsides (poor performance, single point of failure, complexity of the overall solution).

Ignite 2.1 solves all these problems with a single line of configuration:

```cs
var cfg = new IgniteConfiguration { PersistentStoreConfiguration = new PersistentStoreConfiguration() };
```