---
layout: post
title: What's new in Apache Ignite.NET 2.13
---

[Apache Ignite](https://ignite.apache.org/) 2.13 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0).
Calcite-based SQL engine is the highlight of the release, but we are here to talk about .NET side of things, where thin client got some more improvements: retry policy, heartbeat messages, and more. 

(There was no blog post about Ignite [2.12](https://blogs.apache.org/ignite/entry/apache-ignite-2-12-0) because it was focused on bugfixes).

# Thin Client Keep-Alive (Heartbeats)

Periodic heartbeat messages were introduced to improve thin client connection robustness, especially in long-living scenarios.

Those messages are sent in background when the client is idle (defined by `HeartbeatInterval` config property), so that connection loss can be detected early.

We can use the following code to demonstrate:

```csharp
var cfg = new IgniteClientConfiguration("127.0.0.1")
{
    EnableHeartbeats = true,
    HeartbeatInterval = TimeSpan.FromMilliseconds(300),
    Logger = new ConsoleLogger { MinLevel = LogLevel.Debug }
};

var client = Ignition.StartClient(cfg);

Console.WriteLine("Client connected.");

while (true)
{
    Console.WriteLine("Press any key to perform a request...");
    Console.ReadKey();

    Console.WriteLine("Cluster node: " + client.GetCluster().GetNode());
}
```

* Start Ignite server in Docker with `docker run -p 10800:10800 apacheignite/ignite`.
* Run the code above.
* While it is waiting for input, stop the server and start it again. Connection loss should be detected and logged.
* Press a key in the program console. Connection will be restored and API call will succeed.

This behavior should be enabled explicitly with `EnableHeartbeats` property.

[IEP-83 Thin Client Keepalive](https://cwiki.apache.org/confluence/display/IGNITE/IEP-83+Thin+Client+Keepalive)

# Thin Client Retry Policy

TODO

[IEP-82 Thin Client Retry Policy](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=195727946)

# Thin Client Service Descriptors

TODO

# SendServerExceptionStackTraceToClient

TODO


# Links

* Main blog post: [https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0](https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0) 
* Full release notes: [https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt](https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt)
* Download: [https://ignite.apache.org/download](https://ignite.apache.org/download.cgi)
