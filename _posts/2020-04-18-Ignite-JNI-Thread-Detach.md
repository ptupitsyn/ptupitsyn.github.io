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

Modern Java (this has been run on OpenJDK 8+) uses native threads, there is no green thread magic here. So what is going on? How is that possible?


# JNI and Threads

TODO: Measure the cost to call Attach/Detach
GetBenchmark: 160K op/s (6 us per call); with attach/detach on every call - 16K op/s (60 us per call) => attach+detach takes 50us.

# Searching for the Fix