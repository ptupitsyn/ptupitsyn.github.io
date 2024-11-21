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

By stepping through the code in debugger, we can see that the crash occurs at a call to unmanaged function:
```c++
jint JNI_CreateJavaVM(JavaVM **p_vm, JNIEnv **p_env, void *vm_args)
```

Looking up the error code yields `STATUS_STACK_BUFFER_OVERRUN`. Ignite.NET allocates two pointers on the CLR stack and passes them to the JVM, which then writes data to these pointers. 
The error suggests that the JVM writes more data than expected, causing the stack overrun, but that is not the case.

It was time to go deeper: download JDK symbols and sources and step through the JVM code. 
Which brings us to [this call](https://github.com/microsoft/openjdk-jdk11u/blob/release/jdk-11.0.25_9/src/hotspot/cpu/x86/vm_version_x86.cpp#L618):
```c++
  get_cpu_info_stub(&_cpuid_info);
```

The stub points to a function that is [generated from assembly code at runtime](https://github.com/microsoft/openjdk-jdk11u/blob/release/jdk-11.0.25_9/src/hotspot/cpu/x86/vm_version_x86.cpp#L65). 
The assembly code performs a lot of tricks to detect CPU capabilities, it is not easy to follow and impossible to debug. It even deals with 386/486 CPUs, which is quite fascinating:

```c++
    //
    // if we are unable to change the AC flag, we have a 386
    //
    __ xorl(rax, HS_EFL_AC);
    __ push(rax);
    __ popf();
    __ pushf();
    __ pop(rax);
    __ cmpptr(rax, rcx);
    __ jccb(Assembler::notEqual, detect_486);
```

Java is old.
