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

**Partition Awareness**

Initial implementation of Thin Client used a single connection to a given server node to perform all operations. As we know, Ignite distributes cache entries evenly across cluster nodes. When Thin Client is connected to node A, but requested entry is on node B, an additional network request has to be made from A to B.

Ignite 2.8 introduces Thin Client Partition Awareness feature: thin clients can connect to all server nodes, determine primary node for the given key, and route requests directly to that node, avoiding extra network hops. This routing is very quick, it involves some basic math on the key hash code to determine target node according to known partition distribution.

TODO: Benchmark results here


---------------

TODO:
- Recap of changes since 2.1
- Explain Thin Client briefly
- Partition Awareness and Failover
- Explain cross-platform support along with .NET Core 3.x support (Linux, macOS, Mono), no more C++ layer and extra DLLs, no more powershell scrips, publish works, still supports .NET 4.0
- LINQ: DML updates, RegEx, local collection join
- Dynamic service proxies