---
layout: post
title: Distributed Caching in ASP.NET Core with Apache Ignite 3
date: 2025-11-25
author: Pavel Tupitsyn
categories: [Apache Ignite, .NET, Compute]
---

TODO: DO NOT PUBLISH, UNUSABLE WITHOUT https://issues.apache.org/jira/browse/IGNITE-23973

Distributed caching lets multiple app instances share a single cache store so data is not duplicated and all instances observe a consistent state. 
In this post, we explore how to set up distributed caching in ASP.NET Core applications using Apache Ignite 3.

# What is Distributed Caching?

A simple caching mechanism stores data in memory to improve performance and reduce the load on the underlying data source.
However, with multiple instances of an application, each instance has its own separate cache:
* Each instance must fetch data from the data source to cache it
* Data is duplicated in each instance
* Inconsistencies can arise due to failover and load balancing when different instances have different cached data

```text
                +----------------------+
                | Users / Load Balancer|
                +----------+-----------+
                           |
        -------------------+-------------------
        |                  |                  |
+-------v-------+  +-------v-------+  +-------v-------+
| App Instance 1|  | App Instance 2|  | App Instance 3|
+-------+-------+  +-------+-------+  +-------+-------+
        |                  |                  |
 +------v------+    +------v------+    +------v------+
 | In-Memory   |    | In-Memory   |    | In-Memory   |
 | Cache 1     |    | Cache 2     |    | Cache 3     |
 +-------------+    +-------------+    +-------------+
```

A [distributed cache](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed?view=aspnetcore-10.0) is shared:
* One app instance fetches the data, others can read it
* Data is stored only once
* All instances see the same state
* Cache survives instance restarts

```text
                +----------------------+
                | Users / Load Balancer|
                +----------+-----------+
                           |
        -------------------+-------------------
        |                  |                  |
+-------v-------+  +-------v-------+  +-------v-------+
| App Instance 1|  | App Instance 2|  | App Instance 3|
+-------+-------+  +-------+-------+  +-------+-------+
        |                  |                  |
        |----------------- |----------------- |
                           |
                    +------v------+
                    | Distributed |
                    | Cache       |
                    +-------------+
```

[Apache Ignite](https://ignite.apache.org/) is perfect for distributed caching:
* In-memory storage for low latency
* Fast key-value API
* Easy horizontal scalability

With the [Apache.Extensions.Caching.Ignite](https://www.nuget.org/packages/Apache.Extensions.Caching.Ignite#readme-body-tab) package, 
integrating Ignite distributed cache in ASP.NET Core apps is straightforward.


# Walkthrough

Full source code is available on [GitHub](https://github.com/ptupitsyn/ignite3-dotnet-distributed-cache)

- Create a new ASP.NET Core project: ```dotnet new webapi```
- Add Ignite package: ```dotnet add package Apache.Extensions.Caching.Ignite```
- Configure Ignite distributed cache in `Program.cs`:
```csharp
builder.Services
    .AddIgniteClientGroup(new IgniteClientGroupConfiguration
    {
        ClientConfiguration = new IgniteClientConfiguration("localhost")
    })
    .AddIgniteDistributedCache(options => options.TableName = "ASPNET_DISTRIBUTED_CACHE");
```
- Inject `IDistributedCache` and use to store/retrieve data:
```csharp
app.MapGet("/weatherforecast",  async (IDistributedCache cache) =>
    {
        const string cacheKey = "weather_forecast";

        byte[]? cachedData = await cache.GetAsync(cacheKey);
        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<IList<WeatherForecast>>(cachedData);
        }

        IList<WeatherForecast> forecast = FetchForecast();

        cachedData = JsonSerializer.SerializeToUtf8Bytes(forecast);
        await cache.SetAsync(cacheKey, cachedData, CancellationToken.None);

        return forecast;
    });
```

# In-Memory Ignite Tables

By default, Ignite uses disk-based `aipersist` storage engine for tables, and `IgniteDistributedCache` creates a table with the default storage profile.

We typically don't need persistence for caching, so let's create an in-memory table for better performance.

```sql
DROP TABLE IF EXISTS ASPNET_DISTRIBUTED_CACHE;
CREATE ZONE IF NOT EXISTS inmem_zone STORAGE PROFILES ['inmem'];
CREATE TABLE ASPNET_DISTRIBUTED_CACHE (key VARCHAR PRIMARY KEY, val VARBINARY) ZONE inmem_zone STORAGE PROFILE 'inmem';
```

With this, `IgniteDistributedCache` will use the in-memory table for cache entries.

# Conclusion

Distributed caching is essential for modern scalable ASP.NET Core applications, and Apache Ignite 3 provides a high-performance solution.

# Links

* [Apache Ignite](https://ignite.apache.org/)
* [Apache.Extensions.Caching.Ignite](https://www.nuget.org/packages/Apache.Extensions.Caching.Ignite)
* [Demo Source Code](https://github.com/ptupitsyn/ignite3-dotnet-distributed-cache)
* [ASP.NET Core Distributed Caching Documentation](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed?view=aspnetcore-10.0)
