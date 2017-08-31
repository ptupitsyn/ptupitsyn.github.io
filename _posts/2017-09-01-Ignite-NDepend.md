---
layout: post
title: Analysing Ignite.NET Code With NDepend
---

![LINQPad Logo](../images/ignite-ndepend.png)

Along with unit testing, continuous integration, and code review, static code analysis is invaluable for maintaining healthy code base.

[Ignite.NET](https://github.com/apache/ignite/tree/master/modules/platforms/dotnet) uses [FxCop](https://en.wikipedia.org/wiki/FxCop) and [ReSharper](https://www.jetbrains.com/resharper/) code analysis from early days, and this process is included in continuous integration on [Ignite TeamCity](https://ci.ignite.apache.org/viewType.html?buildTypeId=Ignite20Tests_IgnitePlatformNetInspections), so I was fairly confident that our code is (mostly) fine.

Not so long ago [Patrick Smacchia](https://blog.ndepend.com/author/psmacchia/) reached out to me and suggested to try [NDepend](http://www.ndepend.com/) as well. Let's see how it works out!

# Getting Started with NDepend

# Exploring critical issues