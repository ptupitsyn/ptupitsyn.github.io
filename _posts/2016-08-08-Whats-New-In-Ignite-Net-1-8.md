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

In previous Ignite.NET versions all logging happened in Java, there was no possibility to configure logging or write to the log on .NET side.

New version [provides](https://apacheignite-net.readme.io/docs/logging) `ILogger` interface and `IgniteConfiguration.Logger` property to define a custom logger,
and there are `ILogger` implementations for NLog and log4net.
When `IgniteConfiguration.Logger` is not specified (default), old behavior with Java logging is in effect.
When the property is set, all logs from Java and .NET are redirected to the specified implementation.

You can also write to the log (no matter if custom logger is configured) via `IIgnite.Logger`.
`ILogger` interface has only one logging method with all the details.
Helper extension methods with simpler signatures are in `Apache.Ignite.Core.Log.LoggerExtensions` class.
This design means that `ILogger` implementors have only two methods to write and don't have to inherit anything.

Here is an example of colorized console output using `IgniteNLogLogger`:

```cs
var logCfg = new LoggingConfiguration();
var target = new ColoredConsoleTarget();
logCfg.AddTarget("con", target);
logCfg.LoggingRules.Add(new LoggingRule("*", NLog.LogLevel.Info, target));

LogManager.Configuration = logCfg;

var cfg = new IgniteConfiguration
{
    Logger = new IgniteNLogLogger(),
    JvmOptions = new [] {"-DIGNITE_QUIET=false"}
};

var ignite = Ignition.Start(cfg);

ignite.Logger.Info("Hello World!");
```

# Transaction Deadlock Detection

Ignite now detects deadlocks in pessimistic transactions automatically when transaction timeout is specified, and throws `TransactionDeadlockException`.
See [documentation](https://apacheignite-net.readme.io/docs/transactions#deadlock-detection-in-pessimistic-transactions) for more details.

# New Examples

Binary and source distributions (available on [ignite.apache.org](https://ignite.apache.org/download.cgi#binaries))
now include a lot more examples in `platforms\dotnet\examples`.

I like these three in particular:

* `TransactionDeadlockDetectionExample` shows how transaction deadlock can be caused, detected, and handled.
* `BinaryModeExample` demonstrates cache and SQL usage without classes, with dynamically generated entities
* `MultiTieredCacheExample` shows how cache entries are stored in heap, on-heap, and swap memory.

# LINQ Improvements

`CompiledQuery` class is deprecated in favor of `CompiledQuery2`, which removes a lot of limitations:

* Skip & Take are supported
* Embedded arguments are allowed (`.Where(x => x.Key > 5)`)
* Argument reuse is allowed (`.Where(x=>x.Key > foo && x.Value.Id > foo)`)

`CompiledQuery2` has a similar API, but it takes `Expression<Func<T1, IQueryable<T>>>` instead of `Func<T1, IQueryable<T>>`, which allows better query inspection.

There is also a new method, `CompiledQueryFunc<T> Compile<T>(IQueryable<T> query)`,
which compiles an existing query to a delegate `CompiledQueryFunc<T>(params object[] args)`.
This method provides a lot of flexibility, because you can build LINQ expression dynamically, and it can have any number of arguments:

```cs
  var nameFilter = "";
  decimal? maxPrice = 20;
  
  var cache = ignite.GetCache<int, Product>("products");		
  var qry = cache.AsCacheQueryable();
  
  if (nameFilter != null)
    qry = qry.Where(x => x.Value.Name.Contains(nameFilter));
    
  if (maxPrice != null)
    qry = qry.Where(x => x.Value.Price <= maxPrice);
    
  var compiledQry = CompiledQuery2
```

The downside here is that arguments should be provided carefully to the compiled delegate in correct order.