---
layout: post
title: Breaking Change in .NET 9 Crashes Ignite Node
---

.NET 9 is out, and it brings a breaking change that causes process crash during Ignite.NET node startup on Windows.

# Investigation

Recently we got a [report](https://lists.apache.org/thread/4080xyvyqljqq0oczj6cf0fskmqjpdzq) from a user that 
Ignite crashes on startup after .NET 9 upgrade with a mysterious error:

```
process 7692 exited with code -1073740791 (0xc0000409)
```

* It works on Linux and crashes on Windows
* The only change is .NET 8 -> .NET 9 upgrade
* The process exits abruptly, there is no stack trace or any other information

By stepping through the code in debugger, we can see that the crash occurs at a call to unmanaged `JNI_CreateJavaVM` function.

Looking up the error code yields `STATUS_STACK_BUFFER_OVERRUN`.
