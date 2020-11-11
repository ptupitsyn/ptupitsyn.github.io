---
layout: post
title: Apache Ignite on .NET 5
---

Exploring .NET 5 with Apache Ignite

# Ignite + .NET 5

Official .NET 5 support will be announced in Ignite 2.10, but preliminary testing shows that everything works as expected on Windows and Linux.

# Top-level statements

Top-level statements reduce boilerplate, so the smallest Ignite.NET program is literally the following:
```cs
Apache.Ignite.Core.Ignition.Start();
```

With separate namespace import and graceful shutdown:
```cs
using Apache.Ignite.Core;
using var ignite = Ignition.Start();
```

This is nice for examples and tutorials, though I'm a bit skeptical about huge amounts of syntactic sugar being added to C#.

# Records

C# 9 [Record Types](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#record-types) are perfect for working with data in Ignite.
Records are immutable, provide value equality, copy constructor, friendly string representation, and they are easy to declare. 

Value equality is particularly handy, because Ignite uses value equality for cache keys. 
TODO: Explain how value equality aligns with Ignite equality

---------

TODO: Records serialization and hash code details
TODO: Top-level program
TODO: Target-typed new for cache ops
