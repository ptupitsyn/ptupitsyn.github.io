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

```
|            Method |     Mean |   Error |
|------------------ |---------:|--------:|
|  StreamThinClient | 106.5 ms | 3.25 ms |
| StreamThickClient | 109.7 ms | 2.19 ms |
```

[(150000 entries, Core i7-9700K, Ubuntu 20.04, .NET 5.0.5).](https://github.com/apache/ignite/blob/850f9bf788de593762184f33420656a890660e2a/modules/platforms/dotnet/Apache.Ignite.BenchmarkDotNet/ThinClient/ThinClientDataStreamerBenchmark.cs)

# Data Streamer API Improvements

Existing data streamer API has been updated. 

* `AddData` and `RemoveData` methods return a `Task` for the current batch, and this task can't be awaited, which is confusing.
Those methods were marked as obsolete. Instead, void `Add` and `Remove` methods were added together with `GetCurrentBatchTask`.
Current batch task is not needed in most use cases, so `AddData` calls can be simply replaced with `Add`.
* `FlushAsync` has been added for non-blocking flush. Note that `Dispose` calls `Flush`, so it is recommended to call `FlushAsync` manually in async scenarios.
* `long AutoFlushFrequency` is replaced with `TimeSpan AutoFlushInterval`.


# .NET 5 Support

[.NET 5](https://dotnet.microsoft.com/download/dotnet/5.0) is now officially supported, including [single-file application](https://docs.microsoft.com/en-us/dotnet/core/deploying/single-file) scenarios.

Here is how to create and pack an Ignite.NET server application into a single executable file, including all the required Ignite files (including jars):
* `mkdir ignite-single-file-test && cd ignite-single-file-test`
* `dotnet new console`
* `dotnet add package Apache.Ignite`
* Add `Apache.Ignite.Core.Ignition.Start();` line to the `Main` method
* `dotnet publish --runtime linux-x64 /p:PublishSingleFile=true /p:IncludeAllContentForSelfExtract=true --self-contained true --output pub` (make sure to fix `runtime` if you are not on Linux, e.g. `win-x64`)

As a result, there is a self-contained `ignite-single-file-test` executable file in the `pub` directory which can be copied to any machine and executed there. The only requirement is a compatible JDK. This approach simplifies deployments, including modern containerized environments.

See [Ignite on .NET 5](2020-11-11-Ignite-on-NET-5.md) post for more info on other .NET 5 features. 

# Reworked Examples

Examples were reworked from .NET Framework to .NET Core:
* Run on any OS.
* Can be run from command line or any IDE (Visual Studio, VS Code, Rider).
* Can be easily installed from NuGet as a `dotnet new` template.

See [README](https://github.com/apache/ignite/blob/master/modules/platforms/dotnet/examples/README.md) for more details.

# Links

* Main blog post: https://blogs.apache.org/ignite/entry/apache-ignite-2-11-stabilization 
* Full release notes: https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt
* Download: https://ignite.apache.org/download.cgi
