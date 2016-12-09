---
layout: post
title: What's new in Apache Ignite.NET 1.8
---

Apache Ignite 1.8 [has been released](https://ignite.apache.org/news.html#release-1.8.0) yesterday. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)

Three big new features are Entity Framework 2nd level cache, ASP.NET session state cache, and custom logging.

# Entity Framework Second Level Cache

Apache Ignite is often used as a middle layer between disk-based RDBMS and the application logic to increase performance.
Common approach is to implement a write-through & read-through [cache store](https://ptupitsyn.github.io/Entity-Framework-Cache-Store/).
However, this approach requires changing existing data layer completely and switching to Ignite APIs.

Now there is a new way to boost database performance in Entity Framework 6.x applications which requires only minimal configuration changes:
[Ignite EF 2nd Level Cache](https://apacheignite-net.readme.io/docs/entity-framework-second-level-cache)

# ASP.NET Session State Cache

ASP.NET stores session state data in memory by default.
For web farms there is SQL Server provider, so that session state is shared between servers.
Ignite offers [distributed in-memory session state storage](https://apacheignite-net.readme.io/docs/aspnet-session-state-caching),
which also shares data between servers, but is faster and more fault tolerant than SQL Server.

# Custom Logging

https://apacheignite-net.readme.io/docs/logging