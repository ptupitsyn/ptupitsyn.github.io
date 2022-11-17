---
layout: post
title: Use Ignite in Rust (or any language) with .NET Native AOT
---

.NET 7 introduces AOT, which allows easy interop with any language on any OS. To demonstrate the possibilities, let's bring some Ignite APIs to Rust!

# Native AOT in .NET

[AOT (ahead-of-time compilation)](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) produces binaries compiled to native code:
* No IL or JIT
* Self-contained (standard library and third-party dependencies are packed into one file)
* No external dependencies

The primary motivation is performance - quick startup, reduced memory usage. Deployment is simpler, too.

But it can also [produce native libraries](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/#build-native-libraries) - `.dll` for Windows, `.so` for Linux, `.dylib` for macOS.
And those libraries are also self-contained, you just copy one file anywhere and use it.

# Ignite and Rust

If I want to use Ignite from Rust, where we don't have a native client, I have a few options:
* [REST API](https://ignite.apache.org/docs/latest/restapi)
* Interop with [C++ client](https://ignite.apache.org/docs/latest/quick-start/cpp) 
* [Thin client protocol](https://cwiki.apache.org/confluence/display/IGNITE/IEP-9+Thin+Client+Protocol)
* [JNI](https://en.wikipedia.org/wiki/Java_Native_Interface)

REST and C++ clients are [somewhat limited](https://cwiki.apache.org/confluence/display/IGNITE/Thin+clients+features) and don't provide features like 
[Data Streamer](https://ignite.apache.org/docs/latest/data-streaming), [Continuous Queries](https://ignite.apache.org/docs/latest/key-value-api/continuous-queries) and others.
Implementing thin client protocol from scratch is not easy, and JNI is very hard to work with.

This brings us to .NET. Ignite.NET client is fully featured and well optimized. 
With AOT we should be able to build a native library that exposes any APIs we need and call them from Rust.

# Building Native .NET Library

1. Create a classlib project: `dotnet new classlib`
2. Install Ignite: `dotnet add package Apache.Ignite --version 2.15.0-alpha202211` (we have to use a pre-release version because of a small [bugfix](https://github.com/apache/ignite/commit/6ad8d4085b48f0bd667f478df7a1b91e521c97c3) that enables AOT and is not yet released)
3. Enable AOT: add `<PublishAot>true</PublishAot>` to the csproj 
4. Expose Ignite APIs with [UnmanagedCallersOnly attribute](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.unmanagedcallersonlyattribute?view=net-6.0):
```csharp
public static class Exports
{
    [UnmanagedCallersOnly(EntryPoint = "CachePut")]
    public static void CachePut(int key, int val) => Cache.Put(key, val);

    [UnmanagedCallersOnly(EntryPoint = "CacheGet")]
    public static int CacheGet(int key) => Cache.Get(key);

    private static ICacheClient<int, int> Cache => Client.Value.GetOrCreateCache<int, int>("c");
}
```

(full source code is available on GitHub: https://github.com/ptupitsyn/ignite-net-rust-interop/tree/main/dotnet/libignite)

Now we can publish it and produce a self-contained native `.so` file (I'm using Linux here):
```
dotnet publish --configuration Release --runtime linux-x64 --output publish
```

Then verify exported symbols with `nm -gD libignite.so`, which shows something like this:
```
...
                 U bsearch
0000000000971350 T CacheGet
0000000000971300 T CachePut
                 U calloc
...
```

# Calling .NET Library from Rust

Nothing special here, just make sure that the library file has `lib` prefix (`libignite.so` in our case), Rust seems to require it for some reason.

1. Create a project: `cargo init`
2. Add `build.rs` to link the library:
```rust
fn main() {
    println!("cargo:rustc-link-search=native=../../dotnet/libignite/publish");
    println!("cargo:rustc-link-lib=ignite");
}
```

TODO:
* C++ client can be used too, but it has a lot less features (link comparison page).
* Put code in a repo with instructions and requirements.
* Build and run with Docker to prove there are no dependencies?
