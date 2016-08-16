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


## 