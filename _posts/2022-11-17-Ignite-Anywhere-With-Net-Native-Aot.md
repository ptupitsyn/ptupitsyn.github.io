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

This brings us to .NET.



TODO:
* C++ client can be used too, but it has a lot less features (link comparison page).
* Put code in a repo with instructions and requirements.
* Build and run with Docker to prove there are no dependencies?
