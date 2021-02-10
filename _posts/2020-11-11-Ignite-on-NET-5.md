---
layout: post
title: Apache Ignite on .NET 5
header-img: "https://raw.githubusercontent.com/ptupitsyn/ptupitsyn.github.io/master/images/ignite_logo.png"
tags: [ignite, apache, C#, F#, .NET, performance]
---

.NET 5 was released on November 10. Preliminary testing shows that Ignite.NET works as expected with the new SDK and all tests pass, except single-file deployment on Linux (see below).
Official support for .NET 5 is coming in Ignite 2.10. 

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

Value equality is particularly handy, because Ignite compares cache keys based on field values,
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
Here, `key1` and `key2` are two different instances of the same class, so `ReferenceEquals` returns `false`, but `==` and `Equals` return `true`,
because all property values are equal. And Ignite considers them equal, too: value can be retrieved correctly by key.

Note how another C# 9 feature, [target-typed new](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#fit-and-finish-features), is used here to create keys and values in a more concise way.


# Single-file applications

[Single-file deployment](https://docs.microsoft.com/en-us/dotnet/core/deploying/single-file) was improved in .NET 5.
Before that, .NET Core host used to create a temporary directory and extract all application files there - it was not really a single-file app, but a self-extracting app. Ignite works fine in that old mode.

New mode, however, is a true single-file app where everything is loaded directly, without temp files. And, unfortunately, this is where Ignite fails on Linux.
If we publish a simple Ignite app like this `dotnet publish -c Release --self-contained true -r linux-x64 -p:PublishSingleFile=true` and run it, we get an error:

```
Unhandled exception. System.DllNotFoundException: Unable to load shared library 'libcoreclr.so' or one of its dependencies. In order to help diagnose loading problems, consider setting the LD_DEBUG environment variable: liblibcoreclr.so: cannot open shared object file: No such file or directory
   at Apache.Ignite.Core.Impl.Unmanaged.Jni.DllLoader.NativeMethodsCore.dlopen(String filename, Int32 flags)
   at Apache.Ignite.Core.Impl.Unmanaged.Jni.DllLoader.Load(String dllPath)
   at Apache.Ignite.Core.Impl.Unmanaged.Jni.JvmDll.LoadDll(String filePath, String simpleName)
   at Apache.Ignite.Core.Impl.Unmanaged.Jni.JvmDll.Load(String configJvmDllPath, ILogger log)
   at Apache.Ignite.Core.Ignition.Start(IgniteConfiguration cfg)
   at Apache.Ignite.Core.Ignition.Start()
   at <Program>$.<Main>$(String[] args)
```

`libcoreclr.so` is used for a couple of unmanaged calls: `dlopen` (to load the JVM) and `pthread_key_create` ([to clean up JNI threads](https://ptupitsyn.github.io/Ignite-JNI-Thread-Detach/)).

This can be worked around by redirecting unmanaged calls to `libdl.so` with the following code before `Ignition.Start` call:

```cs
bool libdlLoaded = false;

AssemblyLoadContext.Default.ResolvingUnmanagedDll += (assembly, lib) =>
{
    if (assembly == typeof(Ignition).Assembly)
    {
        if (lib == "libcoreclr.so")
        {
            if (!libdlLoaded)
            {
                libdlLoaded = true;
                return NativeLibrary.Load("libdl.so");
            }

            return NativeLibrary.Load("libpthread.so");
        }
    }

    return IntPtr.Zero;
};
```

Alternative fix:

```cs
NativeLibrary.SetDllImportResolver(
    typeof(Ignition).Assembly,
    (libraryName, _, _) => libraryName == "libcoreclr.so"
        ? (IntPtr) (-1)
        : IntPtr.Zero);
```

# Performance

.NET 5 brings [a long list of internal performance improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-5/), so existing code becomes faster for free.

You can expect performance improvements in .NET-specific Ignite features, such as Platform Cache (see more details in the [Ignite 2.9 post](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.9/)):  

```
|                 Method |       Runtime |      Mean | Ratio |
|----------------------- |-------------- |----------:|------:|
| ComputeSumWithPlatform | .NET Core 3.1 | 10.603 ms |  1.00 |
| ComputeSumWithPlatform | .NET Core 5.0 |  8.612 ms |  0.81 |
```
