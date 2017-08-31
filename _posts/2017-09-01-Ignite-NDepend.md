---
layout: post
title: Analysing Ignite.NET Code With NDepend
---

![LINQPad Logo](../images/ignite-ndepend.png)

Along with unit testing, continuous integration, and code review, static code analysis is invaluable for maintaining healthy code base.

[Ignite.NET](https://github.com/apache/ignite/tree/master/modules/platforms/dotnet) uses [FxCop](https://en.wikipedia.org/wiki/FxCop) and [ReSharper](https://www.jetbrains.com/resharper/) code analysis from early days, and this process is included in continuous integration on [Ignite TeamCity](https://ci.ignite.apache.org/viewType.html?buildTypeId=Ignite20Tests_IgnitePlatformNetInspections), so I was fairly confident that our code is (mostly) fine.

Not so long ago [Patrick Smacchia](https://blog.ndepend.com/author/psmacchia/) reached out to me and suggested to try [NDepend](http://www.ndepend.com/) as well. Let's see how it works out!

# Getting Started with NDepend

We are going to work only with `Apache.Ignite.Core` project: it is the biggest, most important, and most complicated part of Ignite.NET.

NDepend, like FxCop, operates on a built assembly (dll file), so we have to build the project in Visual Studio, 
open up the assembly in `VisualNDepend.exe`, and hit F5 (Run Analysis). The process is surprisingly quick: 1 second on my machine,
where ReSharper (command-line) and FxCop take several seconds.

Upon analysis completion we are presented with some statistics and a summary of issues:

![NDepend Dashboard](../images/NDepend/dashboard.png)

2654 issues! Whoops! No blockers at least.
But every static analyzer produces false positives and irrelevant issues, so it's not time to worry, yet.

# Exploring Critical Issues

There are 4 critical issues and 7 critical rule violations, we are going to start with these.
Click on red `4` number to reveal a list of issues:

![NDepend Critical Issues](../images/NDepend/criticals.png)

Hovering over the rule name shows a pop-up window with [detailed rule description](http://www.ndepend.com/default-rules/Q_Avoid_types_initialization_cycles.html) along with possible false positive scenarios.

Our case seems like a false positive, since [BinaryObjectBuilder](https://github.com/apache/ignite/blob/82e5f8a6553323e793c01c54e24dda6d47188ce6/modules/platforms/dotnet/Apache.Ignite.Core/Impl/Binary/BinaryObjectBuilder.cs) does not have a static constructor and does not use [BinarySystemHandlers](https://github.com/apache/ignite/blob/82e5f8a6553323e793c01c54e24dda6d47188ce6/modules/platforms/dotnet/Apache.Ignite.Core/Impl/Binary/BinarySystemHandlers.cs) in static field initializers.

The only usage of `BinarySystemHandlers` is on [Line 127](https://github.com/apache/ignite/blob/82e5f8a6553323e793c01c54e24dda6d47188ce6/modules/platforms/dotnet/Apache.Ignite.Core/Impl/Binary/BinaryObjectBuilder.cs#L127), but this has nothing to do with type initialization.

However, looking closer at how these three classes (`BinaryUtils`, `BinaryObjectBuilder`, `BinarySystemHandlers`) relate, reveals a real code quality issue: method `GetTypeId` does not really belong in `BinarySystemHandlers`, it operates only on some constants from `BinaryUtils`. So we could move `GetTypeId` to `BinaryUtils`, but that class is already ugly ("utility class anti-pattern"). Proper solution is to [extract type code handling logic to a separate class](https://issues.apache.org/jira/browse/IGNITE-6233).