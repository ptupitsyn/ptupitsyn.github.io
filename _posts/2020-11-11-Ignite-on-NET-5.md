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

```cs
using System;
using Apache.Ignite.Core;

using var ignite = Ignition.Start();
var cache = ignite.GetOrCreateCache<EmployeeKey, Employee>("employee");

EmployeeKey key1 = new(2, "b-1");
EmployeeKey key2 = new(2, "b-1");

cache[key1] = new("John Doe", new DateTime(2016, 1, 1));

Console.WriteLine($"Value from cache: {cache[key2]}");
Console.WriteLine($"Equals: {key1 == key2}, ReferenceEquals: {ReferenceEquals(key1, key2)}");

public sealed record Employee(string Name, DateTime StartDate);

public sealed record EmployeeKey(int CompanyId, string Id);
```

Records are [reference types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/reference-types).
Here, `key1` and `key2` two different instances of the same class, so `ReferenceEquals` returns `false`, but `==` and `Equals` return `true`,
because all property values are equal.

And Ignite considers them equal, too: value from cache can be retrieved correctly. Ignite uses it's own mechanism to determine equality, which is also value-based.


---------

TODO: Records serialization and hash code details
TODO: Top-level program
TODO: Target-typed new for cache ops
