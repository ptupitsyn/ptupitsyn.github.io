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
See [Getting Started](https://apacheignite-net.readme.io/docs/getting-started-2#section-using-app-config).

## Forward Java output to the .NET console

Ignite writes useful information to console: topology snapshots, metrics, errors and warnings, etc. This information is also written to the log, but console may be more handy during development.

Most of this information is written from underlying Java code. This is fine when you run a console application. However, this does not work with tools that intercept .NET console: unit test runners (Visual Studio and ReSharper), LINQPad, etc.

Ignite 1.7 has this fixed. All console output is now written using native .NET mechanism, and you can expect any tooling to work properly. Below is a screenshot of Ignite running in LINQPad.
![Ignite in LINQPad](../images/2016-08-08-Whats-New-In-Ignite-Net-1-7.1/linqpad-output.png)

## Add Java stack trace on .NET side in IgniteException.InnerException

## Configuration schema NuGet package

## ASP.NET Output Cache provider