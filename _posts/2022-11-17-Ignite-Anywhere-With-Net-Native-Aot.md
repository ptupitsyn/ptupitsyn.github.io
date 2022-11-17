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

# Build Native .NET Library

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

# Call .NET Library from Rust

Nothing special here, just make sure that the library file has a proper ["soname"](https://en.wikipedia.org/wiki/Soname) with `lib` prefix (`libignite.so` in our case).

1. Create a project: `cargo init`
2. Add `build.rs` to link the library:
```rust
fn main() {
    println!("cargo:rustc-link-search=native=../../dotnet/libignite/publish");
    println!("cargo:rustc-link-lib=ignite");
}
```
3. Declare extern functions in `main.rs`:
```rust
extern {
    fn CacheGet(key: i32) -> i32;
    fn CachePut(key: i32, val: i32);
}
```
4. Use them (requires unsafe block):
```rust
let key = 42;
CachePut(key, key + 1);

let res = CacheGet(key);
println!("Result from cache: {}", res);
```

(full source code is at https://github.com/ptupitsyn/ignite-net-rust-interop/tree/main/rust/ignite-client-test)

That's it! Now start an Ignite server node with `docker run -p 10800:10800 apacheignite/ignite`, and run the app, the output is:
```
[20:45:51] [Debug] [ClientSocket] Socket connection attempt: 127.0.0.1:10800
[20:45:51] [Debug] [ClientSocket] Socket connection established: 127.0.0.1:57860 -> 127.0.0.1:10800
[20:45:51] [Debug] [ClientSocket] Handshake completed on 127.0.0.1:10800, protocol version = 1.7.0
[20:45:51] [Debug] [ClientFailoverSocket] Server binary configuration retrieved: BinaryConfigurationClientInternal [CompactFooter=True, NameMapperMode=BasicFull]
Result from cache: 43
```

We can notice that the resulting app is quite fast, `time ./ignite-client-test` shows `real	0m0,047s` on my machine, and this combines app startup, cluster connection, and data exchange.

# Try It

TODO: Script in repo root

# Conclusion

The example above is simplistic, of course - in a real-world scenario we would have to deal with complex data types, shared memory, resource lifetimes, and so on.

However, by reusing Ignite.NET client, we avoid dealing with much bigger complexity of communicating with the cluster of multiple nodes with failover, partition awareness, cluster discovery, async network request handling and multiplexing.
