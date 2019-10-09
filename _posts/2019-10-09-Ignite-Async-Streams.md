---
layout: post
title: Playing with C# 8.0 Async Streams in Apache Ignite
---

C# 8.0 introduces [Asynchronous Streams](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams), which combine lazy enumeration and `async`/`await`. The most obvious use case here are database queries, where every individual record is pulled asynchronously from a remote server. This applies to [Apache Ignite](https://ignite.apache.org/) too - [SQL](https://apacheignite-net.readme.io/docs/sql-queries) and [Scan](https://apacheignite-net.readme.io/docs/cache-queries#scan-queries) query APIs can be updated with async versions. This requires code modification and we can expect those things in future Ignite versions. However, there is one more query type that we can convert to async version right now - [Continuous Query](https://apacheignite-net.readme.io/docs/continuous-queries).

![TODO](../images/TODO)

# TODO

