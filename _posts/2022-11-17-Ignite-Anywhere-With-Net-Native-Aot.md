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

TODO Library



TODO:
* C++ client can be used too, but it has a lot less features (link comparison page).
* Put code in a repo with instructions and requirements.
