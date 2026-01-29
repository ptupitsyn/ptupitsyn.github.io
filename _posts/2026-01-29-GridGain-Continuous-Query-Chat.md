---
layout: post
title: Simplest Chat App Ever with GridGain Continuous Query
date: 2025-11-03
author: Pavel Tupitsyn
categories: [GridGain, .NET, ContinuousQuery]
---

Scalable distributed chat app in just 13 lines of code? Yes, please!

# Show Me the Code

Put the code below into a `chat.cs` file and run it with `dotnet run chat.cs <your-name>` (requires .NET 10):

```csharp
#:package GridGain.Ignite@9.1.16
using Apache.Ignite;
using Apache.Ignite.Table;

using var client = await IgniteClient.StartAsync(new("localhost"));
await client.Sql.ExecuteScriptAsync("CREATE TABLE IF NOT EXISTS CHAT(id uuid PRIMARY KEY, msg varchar)");
var table = (await client.Tables.GetTableAsync("CHAT"))!.RecordBinaryView;

_ = Task.Run(async () =>
{
    await foreach (var msg in table.QueryContinuouslyAsync().SelectMany(x => x.Events))
        Console.WriteLine(msg.Entry![1]);
});

while (true)
    await table.InsertAsync(null, new IgniteTuple { ["id"] = Guid.NewGuid(), ["msg"] = $"{args[0]}: {Console.ReadLine()}" });
```

Run multiple instances of the app in different terminal windows to test it out.

# How It Works

[Continuous Query](https://www.gridgain.com/docs/gridgain9/latest/developers-guide/continuous-queries) is at the heart of this app:
* Guarantees exactly-once delivery of change events.
* Can return previous events (show chat history on startup).

`QueryContinuouslyAsync` returns an `IAsyncEnumerable` of change events happening in the table. `IAsyncEnumerable` is one of my favorite abstractions. It is so simple and powerful:
* Clean and easy to use.
* Non-blocking.
* Natural cancellation (`break`).
* Backpressure is built in: if the consumer is slow, the polling pauses.
* Native LINQ support (like `SelectMany` above to unroll batches).

# Scalability

This toy app can actually handle thousands of users and millions of messages per minute without any changes, 
given enough cluster resources, thanks to GridGain's distributed architecture:
* Automatic partitioning (sharding) of data across cluster nodes.
* Client partition awareness (direct requests to the primary replica).
* Lightweight, persistent, multiplexed client connections (one node can handle [hundreds of thousands of clients](https://ignite.apache.org/blog/apache-ignite-3-client-connections-handling.html)).
* Efficient continuous query mechanism with built-in batching and flow control.
