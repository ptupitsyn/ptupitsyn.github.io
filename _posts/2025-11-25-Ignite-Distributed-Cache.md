---
layout: post
title: Distributed caching in ASP.NET Core with Apache Ignite 3
date: 2025-11-03
author: Pavel Tupitsyn
categories: [Apache Ignite, .NET, Compute]
---

TBD
- Use in-memory table
- Set up distrib cache
- Advise on hybrid cache

# What is Distributed Cache?

# Quick Start

- Create a new ASP.NET Core project ```dotnet new webapp -o IgniteDistributedCache```
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
