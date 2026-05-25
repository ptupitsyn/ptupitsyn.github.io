---
layout: post
title: Simplest Chat App Ever with GridGain Continuous Query
date: 2026-05-25
author: Pavel Tupitsyn
categories: [GridGain, .NET, ContinuousQuery, SQL]
---

[Native AOT deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) is now supported in GridGain .NET client, let's see what can we do with it.

# What is Native AOT?

In short, this mode compiles .NET code to native machine code ahead of time (AOT) instead of just-in-time (JIT).

* Same as C++, Rust, Go, and unlike normal .NET and Java.
* Single binary, no dependencies.
* Faster startup time and lower memory usage.
* Smaller deployment size.