---
layout: post
title: What's new in Apache Ignite.NET 1.7
---

Apache Ignite 1.7 [has been released](https://ignite.apache.org/news.html#release-1.7.0) last week. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)

## Distributed Joins
This is a big one! Previously, SQL joins worked only for colocated data: if cache entry for `John`, who works for `Apache`, is located on node 1, but cache entry for `Apache` is located on node 2, SQL join won't return this pair.

Now, however, this is no longer the issue. Joins work as expected in any scenario.
This feature is not enabled by default, you have to set `SqlQuery.EnableDistributedJoins` or `SqlFieldsQuery.EnableDistributedJoins` to `true` explicitly.

As for LINQ, there is a new overload:

```cs
public static IQueryable<ICacheEntry<K, V>>
    AsCacheQueryable<K, V>(this ICache<K, V> cache, QueryOptions queryOptions)
```

Where `QueryOptions` class has `EnableDistributedJoins` property.

## User-defined AffinityFunction

In addition to predefined `FairAffinityFunction` and `RendezvousAffinityFunction`, it is now possible to create a custom one by implementing `IAffinityFunction` interface or inheriting one of the predefined classes.

## .NET configuration in Apache.Ignite.exe

Apache.Ignite.exe standalone node runner got two new command-line arguments: `ConfigFileName` and `ConfigSectionName`. These can be used to load `IgniteConfigurationSection` either from Apache.Ignite.exe.config, or from any other config file.

## Forward Java output to the .NET console

## Add Java stack trace on .NET side in IgniteException.InnerException

## Configuration schema NuGet package

## ASP.NET Output Cache provider