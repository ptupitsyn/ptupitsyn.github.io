---
layout: post
title: Apache Ignite on .NET 5
---

Notes on .NET 5 and Apache Ignite

# Ignite + .NET 5

.NET 5 was released on November 10. Preliminary testing shows that Ignite.NET works as expected with the new SDK and all tests pass.
We'll announce official support for .NET 5 in Ignite 2.10. 

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

Value equality is particularly handy, because Ignite uses value equality for cache keys,
so with records the comparison behavior is consistent across Ignite operations and regular C# code:

TODO: Example
TODO: Note on attributes?

---------

TODO: Records serialization and hash code details
TODO: Top-level program
TODO: Target-typed new for cache ops
