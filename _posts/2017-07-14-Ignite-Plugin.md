---
layout: post
title: Implementing Ignite.NET Plugin: Distributed Semaphore
---

[Ignite.NET 2.0](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.0/) introduced [plugin system](https://apacheignite-net.readme.io/docs/plugins). Plugins can be .NET-only or .NET + Java. Let's see how to implement the latter.


# Why would I need a plugin?

Ignite.NET is built on top of Ignite (which is written in Java). JVM is started within .NET process, and .NET part talks to Java part and reuses existing Ignite functionality where possible.

Plugin system exposes this platform interaction mechanism to the third parties.
One of the main use cases is making Ignite and third party Java APIs available in .NET.

Good example of such an API is [IgniteSemaphore](https://apacheignite.readme.io/docs/distributed-semaphore), which is not yet available in Ignite.NET.


# Distributed Semaphore API

Ignite Semaphore is very similar to `System.Threading.Semaphore` ([MSDN](https://msdn.microsoft.com/en-us/library/system.threading.semaphore.aspx)), but the effect is cluster-wide: limit the number of threads executing a giving piece of code *across all Ignite nodes*.

It should be used in C# code like this:

```cs
IIgnite ignite = Ignition.GetIgnite();
ISemaphore semaphore = ignite.GetOrCreateSemaphore(name: "foo", count: 3);

semaphore.WaitOne();  // Enter the semaphore (may block)
// Do work
semaphore.Release();
```

Looks simple enough and quite useful; same API as built-in .NET `Semaphore`. Obviously, we can't change `IIgnite` interface, so `GetOrCreateSemaphore` is an extension method. Now onto the implementation!


# Java Plugin

Let's start with Java side of things. We need a way to call `Ignite.semaphore()` method there and provide access to the resulting instance to the .NET platform.