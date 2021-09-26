---
layout: post
title: What's new in Apache Ignite.NET 2.11
---

[Apache Ignite](https://ignite.apache.org/) 2.11 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-11-stabilization).
On .NET side there are new examples, thin client DataStreamer, .NET 5 support, and more.

# Data Streamer in Thin Client

Data streamer API - the most efficient way to load large amounts of data into Ignite - is now available in the .NET thin client. 

* Thin streamer automatically buffers the data and groups it into batches and sends it in parallel to multiple nodes.
* If a server node fails, corresponding operations are retried transparently: at-least-once delivery is guaranteed.

```cs
IIgniteClient client = GetClient();
using (IDataStreamerClient<int, int> streamer = client.GetDataStreamer<int, int>("my-cache"))
{
    streamer.Add(1, 1);
    streamer.Add(2, 2);    

    await streamer.FlushAsync();
}
```

Performance is comparable to the existing thick API:

|            Method |     Mean |   Error | Allocated |
|------------------ |---------:|--------:|----------:|
|  StreamThinClient | 106.5 ms | 3.25 ms |  17.14 MB |
| StreamThickClient | 109.7 ms | 2.19 ms |  13.61 MB |

[(150000 entries, Core i7-9700K, Ubuntu 20.04, .NET 5.0.5).](https://github.com/apache/ignite/blob/850f9bf788de593762184f33420656a890660e2a/modules/platforms/dotnet/Apache.Ignite.BenchmarkDotNet/ThinClient/ThinClientDataStreamerBenchmark.cs)

# Data Streamer API Improvements

Existing data streamer API has been updated. 

* `AddData` and `RemoveData` methods return a `Task` for the current batch, and this task can't be awaited, which is confusing.
Those methods were marked as obsolete. Instead, void `Add` and `Remove` methods were added together with `GetCurrentBatchTask`.
Current batch task is not needed in most use cases, so `AddData` calls can be simply replaced with `Add`.
* `FlushAsync` has been added for non-blocking flush. Note that `Dispose` calls `Flush`, so it is recommended to call `FlushAsync` manually in async scenarios.
* `long AutoFlushFrequency` is replaced with `TimeSpan AutoFlushInterval`.


# .NET 5 Support
TODO

# Reworked Examples
TODO

# Conclusion

TODO: Links





TODO:
* Link to release notes, blog post
  * https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt
  * https://blogs.apache.org/ignite/entry/apache-ignite-2-11-stabilization  
* New examples 
  * https://www.nuget.org/packages/Apache.Ignite.Examples/
  * Usage documented https://github.com/apache/ignite/tree/master/modules/platforms/dotnet/examples
* Thin client streamer
* DataStreamer API improv
* .NET 5 support
* 
