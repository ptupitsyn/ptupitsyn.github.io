---
layout: post
title: Distributed caching in ASP.NET Core with Apache Ignite 3
date: 2025-11-03
author: Pavel Tupitsyn
categories: [Apache Ignite, .NET, Compute]
---

Modern applications run on multiple instances to ensure high availability and scalability. 
A distributed cache is a cache that is shared across all app instances. 
In this post we explore how to set up distributed caching in ASP.NET Core applications with Apache Ignite 3.

# What is Distributed Cache?

A simple caching mechanism stores data in memory to improve performance and reduce the load on the underlying data source.
However, with multiple instances of an application, each instance has its own separate cache:
* Every instance has to fetch data from the data source to cache it
* Data is duplicated in every instance
* Inconsistencies may arise due to failover and load balancing when different instances have different cached data

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

A distributed cache is shared:
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

# Walkthrough

Full source code is available on [GitHub](https://github.com/ptupitsyn/ignite3-dotnet-distributed-cache)

- Create a new ASP.NET Core project ```dotnet new webapi```
- Add Ignite NuGet package ```dotnet add package Apache.Extensions.Caching.Ignite```
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

TBD

```sql
CREATE ZONE IF NOT EXISTS inmem_zone STORAGE PROFILES ['inmem'];

CREATE TABLE ASPNET_CACHE_INMEM (key VARCHAR PRIMARY KEY, val VARBINARY) ZONE inmem_zone STORAGE PROFILE 'inmem';
```

# Hybrid Cache

TBD

# Conclusion

TBD
