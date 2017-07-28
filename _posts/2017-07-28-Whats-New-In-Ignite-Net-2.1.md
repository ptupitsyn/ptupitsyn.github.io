---
layout: post
title: What's new in Apache Ignite.NET 2.1
---

Apache Ignite 2.1 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-1-a) today, introducing Ignite Persistent Store!
Another huge step forward for the project: it becomes a real database with unique feature set, check out the comparison table on [ignite.apache.org frontpage](https://ignite.apache.org/). As usual, we'll have a look at new features from .NET standpoint.

![Apache Ignite Persistent Store](../images/ignite-persistent-store.png)


# Ignite Persistent Store

Ignite has been an in-memory system from the start: RAM is fast, disk is slow, we all know that.
However, in most use cases we still want to persist data in some non-volatile storage, in case of full cluster restart, data center failures, and the like.
Most common solution is an RDBMS serving as [cache store](https://apacheignite-net.readme.io/docs/persistent-store), which has a lot of downsides (poor performance, single point of failure, complexity of the overall solution).

Ignite 2.1 solves all these problems with a single line of configuration, which enables efficient automatic persistence of all cached data on disk.
The following code demonstrates how cache data survives cluster restart (just run it repeatedly):

```cs
var cfg = new IgniteConfiguration 
{ 
    PersistentStoreConfiguration = new PersistentStoreConfiguration() 
};
using (var ignite = Ignition.Start(cfg))
{
    ignite.SetActive(true);  // Required with enabled persistence.
    var cache = ignite.GetOrCreateCache<Guid, string>("myCache");
    cache[Guid.NewGuid()] = "Hello, world!";
    Console.WriteLine("\nCache size: " + cache.GetSize());
}
```

Have you ever seen a database that is so simple to use? I haven't.
There is a single NuGet package behind this code, nothing else. And you can run SQL, LINQ and full-text queries over arbitrary data.

By default everything is stored in Ignite work directory, which happens to be in system Temp folder, so you may want to change this (call `IIgnite.GetConfiguration().WorkDirectory` to see where is it on your machine).

Every Ignite node persists only a part of data which is primary or backup for that node, so storage space and IO load are split between all machines (in contrast with RDBMS cache store).

See more details in [documentation](https://apacheignite.readme.io/docs/distributed-persistent-store); working [example](https://github.com/apache/ignite/blob/master/modules/platforms/dotnet/examples/Apache.Ignite.Examples/Datagrid/StoreExample.cs) can be found in full [binary or source distribution](https://ignite.apache.org/download.cgi).


# Standalone NuGet Deployment

Ignite nodes can be started from code (`Ignition.Start()`) or with a standalone executable (`Apache.Ignite.exe`). However, standalone executable was only available as a part of [full binary distribution](https://ignite.apache.org/download.cgi).

2.1 fixes this discrepancy and includes standalone executable in [Apache.Ignite NuGet package](https://www.nuget.org/packages/Apache.Ignite/) so you can easily install and run Ignite with command line:

```shell
# Powershell
wget https://dist.nuget.org/win-x86-commandline/latest/nuget.exe -OutFile nuget.exe
nuget install Apache.Ignite
.\Apache.Ignite.*\lib\net40\Apache.Ignite.exe
```


# LINQ Improvements

**Conditional data removal** (SQL `DELETE FROM ... WHERE ...`) is now possible:

```cs
var cache = ignite.GetCache<int, Deal>("deals").AsCacheQueryable();

cache.Where(p => p.Value.Company == "Foo").RemoveAll();
```

This is more efficient than loading relevant entries and removing them afterwards.

**Local collection joins** provide efficient alternative to `Contains` and other similar cases:

```cs
```