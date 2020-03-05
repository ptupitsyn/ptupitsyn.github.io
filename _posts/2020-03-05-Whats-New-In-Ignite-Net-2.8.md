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




TODO:
- Recap of changes since 2.1
- Explain Thin Client briefly
- Partition Awareness and Failover
- Explain cross-platform support along with .NET Core 3.x support (Linux, macOS, Mono), no more C++ layer and extra DLLs, no more powershell scrips, publish works, still supports .NET 4.0
- LINQ: DML updates, RegEx, local collection join
- Dynamic service proxies