---
layout: post
title: What's new in Apache Ignite.NET 2.8
---

Thin Client improvements, better cross-platform support, and more!


# Welcome Back

It has been a long time since the [last post in the series](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.1/), and a long time since the last major Ignite release. In this post I would like to catch up on all the improvements on .NET side of things.


# Thin Client and Partition Awareness

From the very beginning, Ignite supports [Client and Server connection modes](https://apacheignite.readme.io/docs/clients-vs-servers). However, Client mode is still relatively "heavy", even though it does not store data and does not perform computations. Starting Ignite.NET client node requires embedded JVM startup, can take a second or more, and consumes a few megabytes of RAM.

This may be undesirable in many use cases, such as short-lived apps, low-powered client machines, CLI tools, and so on. A lightweight thin client protocol was added in Ignite 2.4 to handle those use cases. Quick comparison:

|               | Thick Client        | Thin Client |
|---------------|---------------------|-------------|
| Startup Time  | 1300 ms             | 15 ms       |
| RAM usage     | 40 MB (.NET + Java) | 70 KB       |
| Requires Java | Yes                 | No          |

Ignite.NET thin client is started with `Ignition.StartClient()` call and provides a similar set of APIs. Root interfaces are separate (`IIgnite` -> `IIgniteClient`, `ICache` -> `ICacheClient`), but methods are named in the same way, and most of the code can be easily switched from one API to another and back.

Thin client protocol is [open, extensible, and documented](https://cwiki.apache.org/confluence/display/IGNITE/IEP-9+Thin+Client+Protocol#IEP-9ThinClientProtocol-Handshake), which opens the way for clients in other languages, such as [Python, JavaScript, PHP](https://apacheignite.readme.io/docs/thin-clients).

**Partition Awareness**

Initial implementation of Thin Client used a single connection to a given server node to perform all operations. As we know, Ignite distributes cache entries evenly across cluster nodes. When Thin Client is connected to node A, but requested entry is on node B, an additional network request has to be made from A to B.

Ignite 2.8 introduces Thin Client Partition Awareness feature: thin clients can connect to all server nodes, determine primary node for the given key, and route requests directly to that node, avoiding extra network hops. This routing is very quick, it involves some basic math on the key hash code to determine target node according to known partition distribution. [Benchmark](https://github.com/ptupitsyn/IgniteNetBenchmarks/blob/master/IgniteThinClientBenchmark.cs) of `cache.Get` performance with and without `IgniteClientConfiguration.EnablePartitionAwareness = true` setting:

|            Method |     Mean |    Error |   StdDev |
|------------------ |---------:|---------:|---------:|
|               Get | 90.73 us | 2.114 us | 5.892 us |
| GetPartitionAware | 31.56 us | 0.618 us | 1.234 us |

Your mileage will wary depending on cluster topology, network speeds, and cache entry sizes, but the improvement is significant.

**Failover**

Multi-node thin client connection also means that we get failover behavior: if one or more server nodes fail, client switches to other connections automatically.


# Cross-Platform Support: .NET Core, Linux, macOS

Ignite 2.4 brought .NET Core 2.x support, finally shedding Windows-specific C++ layer and switching to pure .NET implementation of the [JNI layer](https://en.wikipedia.org/wiki/Java_Native_Interface), allowing Ignite.NET apps to run on Linux and macOS.

Ignite 2.8 adds official .NET Core 3.x support and can run on any OS supported by the framework, in any mode - Server, Client, Thin Client.

We have improved the way `.jar` files are handled within NuGet package. Post-build scripts are replaced with [MSBuild .targets file](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-dot-targets-files?view=vs-2019), which is more reliable, cross-platform, and works as expected with `dotnet build` and `dotnet publish`: `.jar` files are copied to build and publish directories automatically, resulting in a self-contained package.

Note that minimum system requirements are still the same: .NET 4.0 and Visual Studio 2010. We care about backwards compatibility within a major version (2.x), but you can expect a switch to .NET Standard 2.0 in upcoming Ignite 3.x.


# LINQ: Conditional and Batch Updates

SQL `UPDATE .. WHERE ..` or `DELETE .. WHERE ..` are usually not possible with ORMs and LINQ. We end up fetching entries with `.Where()` and then updating them one by one, which is suboptimal (to say the least) and not elegant. 

Let's say we want to deactivate all users who have not used our website for more than a year:

```cs
ICacheClient<int, Person> cache = client.GetCache<int, Person>("person");

var threshold = DateTime.UtcNow.AddYears(-1);

IQueryable<ICacheEntry<int,Person>> inactiveUsers = cache.AsCacheQueryable()
	.Where(entry => entry.Value.LastActivityDate < threshold);

foreach (var entry in inactiveUsers)
{
	entry.Value.IsDeactivated = true;
	cache[entry.Key] = entry.Value;
}
```

This code potentially loads thousands of matching entries to the local node, wasting memory and stressing the network. And this goes against Ignite colocated processing mantra: [send code to data, not data to code](https://ignite.apache.org/features/collocatedprocessing.html).

The fix is to use SQL instead:

```cs
cache.Query(new SqlFieldsQuery(
	"UPDATE person SET IsDeactivated = true WHERE LastActivityDate < ?", threshold));
```

Simple, concise, and effective: Ignite will send the query to all nodes and perform updates locally for every cache entry, avoiding any data movement between nodes. However, we don't want SQL in C#, we want LINQ, which is compiler-checked, composable, easier to write and read thanks to the IDE completion.

Ignite 2.5 introduced DML updates via LINQ (in addition to deletes in Ignite 2.1):

```cs
cache.AsCacheQueryable()
	.Where(entry => entry.Value.LastActivityDate < threshold)
	.UpdateAll(d => d.Set(person => person.IsDeactivated, true));
```

This will be transformed to the same SQL query that we have above, combining efficiency and LINQ benefits.


# Dynamic Service Proxies

`IServices.GetDynamicServiceProxy()` is a new API that returns a `dynamic` instance, so we don't have to create an interface beforehand to call arbitrary services. Instead of:

```cs
interface ISomeService
{
    int GetId(string data);
}

...
ISomeService proxy = ignite.GetServices().GetServiceProxy<ISomeService>("someService");
var id = proxy.GetId("foo");
```

We can say:

```cs
dynamic proxy = ignite.GetServices().GetDynamicServiceProxy("someService");
var id = proxy.GetId("foo");
```

This can be useful in a number of scenarios like POCs, calling Java services, and so on.

We can go even further and invoke service methods with a string name:

```cs
var methodName = "foo";
var proxy = (DynamicObject) ignite.GetServices().GetDynamicServiceProxy("someService");
proxy.TryInvokeMember(new SimpleBinder(methodName), new object[0], out var result);
```


# Wrap-up

Thin Client protocol and it's implementations in various languages is one of the major directions for Apache Ignite community these days. Partition Awareness is a big milestone. The next one is automatic server node discovery, so we don't have to provide a list of endpoints manually. You can also expect Compute, Services, Transactions (already available in Java Thin), and other APIs to be added to thin clients in the upcoming versions.