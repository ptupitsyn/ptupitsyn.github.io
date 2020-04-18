---
layout: post
title: Fixing JNI Thread Leak in Ignite.NET
---

[Ignite.NET](https://ignite.apache.org) runs in-process JVM (in thick mode) and interacts with it using [JNI](https://en.wikipedia.org/wiki/Java_Native_Interface). Since version 2.4, when Ignite.NET became cross-platform, we had a stealthy and mysterious bug: JVM thread count kept growing, consuming memory, even though actual OS thread count for the process was low.


# The Reproducer

We've got a reproducer from one of our users, and it boiled down to this:

```cs
var ignite = Ignition.Start(nodeConfig);

var cache = ignite.GetOrCreateCache<int, int>("Test");
cache[1] = 1;

while (true)
{
	var thread = new Thread(_ => cache.Get(1));
	thread.Start();
	thread.Join();
}
```

When you run this on Ignite.NET version 2.4 .. 2.7.6, the following happens:

![ignite logo](../images/jni-thread-leak.png)

[VisualVm](https://visualvm.github.io/) is on top, showing 65279 Java threads. [Linux top](https://linux.die.net/man/1/top) fragment is at the bottom, showing 85 OS threads (rightmost `nTH` column).

Memory consumption is not that bad here, but creating new threads becomes slower and slower: here we see 65K threads created in 11 minutes, while after the bugfix the same program can start/stop 65K threads in 30 seconds.

Modern Java (this has been run on OpenJDK 8+) uses native threads, there is no green thread magic here. So what is going on? How is that possible?

Note that manual thread management is a rarity in modern .NET, most people use `ThreadPool` directly or via `Task` APIs, which makes this bug hardly noticeable - threads rarely start and stop. So this took a while to surface.


# JNI and Threads

As mentioned above, Ignite.NET interacts with in-process JVM via JNI. The process [looks like this](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/invocation.html):

```cs
JNI_CreateJavaVM();
// Call Java methods
DestroyJavaVM();
```

However, this only works in one thread, quote:

```
The JNI interface pointer (JNIEnv) is valid only in the current thread. Should another thread need to access the Java VM, it must first call AttachCurrentThread() to attach itself to the VM and obtain a JNI interface pointer. Once attached to the VM, a native thread works just like an ordinary Java thread running inside a native method. The native thread remains attached to the VM until it calls DetachCurrentThread() to detach itself.
```

Already see where this is going?

Ignite starts the JVM once, and almost all APIs are thread-safe. You can (and should) use one Ignite instance from many threads. So, under the covers, Ignite calls `AttachCurrentThread` whenever a Java call is involved. But we don't call `DetachCurrentThread` right away, for performance reasons.


Attach/Detach call pair takes aroung **50μs** (microseconds) on my machine. This may not seem like much, but it is actually very significant. One of our standard `ICache.Get` benchmarks shows **160K op/s** without `Detach`, and only **16K op/s** with `Detach` - 10 times slower.

So Ignite does not call `DetachCurrentThread` right away. We do it only on thread exit.


# Searching for the Fix