---
layout: post
title: Implementing Ignite.NET Plugin
---

[Ignite.NET 2.0](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.0/) introduced [plugin system](https://apacheignite-net.readme.io/docs/plugins). Plugins can be .NET-only or .NET + Java. Let's see how to implement the latter.


# Why would I need a plugin?

Ignite.NET is built on top of Ignite (which is written in Java). JVM is started within .NET process, and .NET part talks to Java part and reuses existing Ignite functionality where possible.

Plugin system exposes this platform interaction mechanism to the third parties.
One of the main use cases is making Ignite and third party Java APIs available in .NET.

Possible examples:
* ScanQuery with filter builder
* Missing API (??)