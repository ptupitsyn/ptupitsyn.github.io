---
layout: post
title: What's new in Apache Ignite.NET 2.8
---

Thin Client improvements, better cross-platform support, Docker image, and more!


# Welcome Back

It has been a long time since the [last post in the series](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.1/), and a long time since the last major Ignite release. In this post I would like to catch up on all the improvements on .NET side of things.


# Thin Client and Partition Awareness

From the very beginning, Ignite supports [Client and Server connection modes](https://apacheignite.readme.io/docs/clients-vs-servers). However, Client mode is still relatively "heavy", even though it does not store data and does not perform computations. Starting Ignite.NET client node requires embedded JVM startup, can take a second or more, and consumes a few megabytes of RAM.

This may be undesirable in many use cases, such as short-lived apps, low-powered client machines, CLI tools, and so on. A lightweight thin client protocol was added in Ignite 2.4 to handle those use cases. Quick comparison:

|               | Classic Client      | Thin Client |
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


# Cross-Platform Support: .NET Core, Mono, Linux, macOS

Ignite 2.4 brought .NET Core 2.x support, finally shedding Windows-specific C++ layer and switching to pure .NET implementation of the [JNI layer](https://en.wikipedia.org/wiki/Java_Native_Interface), allowing Ignite.NET apps to run on Linux and macOS.

Ignite 2.8 adds official .NET Core 3.x support and can run on any OS supported by the framework, in any mode - Server, Client, Thin Client.

We have improved the way `.jar` files are handled within NuGet package. Post-build scripts are replaced with [MSBuild .targets file](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-dot-targets-files?view=vs-2019), which is more reliable, cross-platform, and works as expected with `dotnet build` and `dotnet publish`: `.jar` files are copied to build and publish directories automatically, resulting in a self-contsined package.


---------------

TODO:
- Recap of changes since 2.1
- Explain Thin Client briefly
- Partition Awareness and Failover
- Explain cross-platform support along with .NET Core 3.x support (Linux, macOS, Mono), no more C++ layer and extra DLLs, no more powershell scrips, publish works, still supports .NET 4.0
- LINQ: DML updates, RegEx, local collection join
- Dynamic service proxies