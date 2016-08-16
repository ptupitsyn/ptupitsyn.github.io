---
layout: post
title: Using Apache Ignite.NET in LINQPad
---

[LINQPad](https://www.linqpad.net/) is a must-have tool for every .NET developer, and it is a great way to explore and try Ignite.NET APIs.

![LINQPad Logo](../images/ignite-linqpad.png)

## Getting Started

1. Download LINQPad: [linqpad.net/Download.aspx](https://www.linqpad.net/Download.aspx). Make sure to choose **AnyCPU** version on x64 OS.
2. Install Ignite.NET NuGet packages:
    * Press F4 (or click Query -> References and Properties menu).
    * Click "Add NuGet...". A warning may appear: `As you don't have LINQPad Premium/Developer Edition, you can only search for NuGet packages that include LINQPad samples.`, this is fine, since Ignite packages do include LINQPad samples.
    * Install packages by clicking "Add To Query" button.
    * Click "Add namespaces" link button and add (at least) the first one: `Apache.Ignite.Core`.
    * Close NuGet window, click OK on Query Properties window.
3. Make sure Language dropdown is set to "C# Expression" (this is the default).
4. Type `Ignition.Start()` (without semicolon) and hit F5.

Ignite node starts and you can see the usual console output in the output pane, as well as the resulting Ignite object dump.

Check out the bundled examples on "Samples" tab to the left.

![Ignite in LINQPad](../images/2016-08-08-Whats-New-In-Ignite-Net-1-7.1/linqpad-output.png)

## Recycling the Worker Process

LINQPad runs user code in a separate process. By default, this process is reused between runs (for performance reasons). 

This has two consequences:

1. Ignite.NET starts an in-process JVM. This JVM is reused when the process is reused, so you can't change the JVM options.
2. `Ignition` class keeps all started nodes in a static map. These nodes will keep running when the process is reused, which may cause `Default Ignite instance has already been started.` exception if you run `Ignition.Start()` code twice.

This behavior may be useful in some scenarios and unwanted in other, but the best thing is that we can control it via built-in `Util.NewProcess` property.
Switch Language dropdown on top to "C# Statement(s)" mode and run the following script:

```cs
Util.NewProcess = true;
Ignition.Start();
```

This script can be run multiple times with no issues, since every time we start from scratch.


## Reusing Started Node

Ignite node takes some time to start (8 seconds on my machine) due to JVM startup and network discovery process.
To quickly iterate on our code in LINQPad, we can reuse started node between runs. For example, the following code reuses started Ignite
instance and reuses existing cache, adding one item per run and showing existing items:

```cs
// Get existing instance or start a new one
var ignite = Ignition.TryGetIgnite() ?? Ignition.Start();

// Get existing cache or create a new one
var cache = ignite.GetOrCreateCache<Guid, DateTime>("cache");

// Add a new entry
cache[Guid.NewGuid()] = DateTime.Now;

// Show all entries
cache.Dump();
```

In contrary to several seconds for a fresh node start, this code runs in a couple of milliseconds. 