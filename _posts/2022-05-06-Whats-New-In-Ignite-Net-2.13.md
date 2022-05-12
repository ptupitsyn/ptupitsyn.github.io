---
layout: post
title: What's new in Apache Ignite.NET 2.13
---

[Apache Ignite](https://ignite.apache.org/) 2.13 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0).
Calcite-based SQL engine is the highlight of the release, but we are here to talk about .NET side of things, where thin client got some more improvements: retry policy, heartbeat messages, and more.

(There was no blog post about Ignite [2.12](https://blogs.apache.org/ignite/entry/apache-ignite-2-12-0) because it was focused on bugfixes).

# Thin Client Keep-Alive (Heartbeats)

Periodic heartbeat messages were introduced to improve thin client connection robustness, especially in long-living scenarios.

Those messages are sent in the background when the client is idle (defined by `HeartbeatInterval` config property) so that connection loss can be detected early.

We can use the following code to demonstrate:

```csharp
var cfg = new IgniteClientConfiguration("127.0.0.1")
{
    EnableHeartbeats = true,
    HeartbeatInterval = TimeSpan.FromMilliseconds(300),
    Logger = new ConsoleLogger { MinLevel = LogLevel.Debug }
};

using var client = Ignition.StartClient(cfg);

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
* Press a key in the program console. Connection will be restored and the API call will succeed.

This behavior should be enabled explicitly with `EnableHeartbeats` property.

In addition, this works together with server-side idle timeouts defined with [ClientConnectorConfiguration.IdleTimeout](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Configuration.ClientConnectorConfiguration.html#Apache_Ignite_Core_Configuration_ClientConnectorConfiguration_IdleTimeout).
When idle timeout is configured on the server, heartbeat interval will be adjusted automatically to `IdleTimeout / 3` so that idle clients are not disconnected by the server.

More details:

* [IEP-83 Thin Client Keepalive](https://cwiki.apache.org/confluence/display/IGNITE/IEP-83+Thin+Client+Keepalive)
* [IgniteClientConfiguration.EnableHeartbeats](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.IgniteClientConfiguration.html#Apache_Ignite_Core_Client_IgniteClientConfiguration_EnableHeartbeats)
* [IgniteClientConfiguration.HeartbeatInterval](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.IgniteClientConfiguration.html#Apache_Ignite_Core_Client_IgniteClientConfiguration_HeartbeatInterval)

# Thin Client Retry Policy

To improve thin client reliability even further, automatic operation retries can be enabled with `RetryPolicy` setting.
If an API call fails due to connection loss, it will be retried transparently.

```cs
var cfg = new IgniteClientConfiguration("127.0.0.1")
{
    Logger = new ConsoleLogger { MinLevel = LogLevel.Debug },
    RetryPolicy = new MyRetryPolicy()
};

using var client = Ignition.StartClient(cfg);

while (true)
{
    Console.WriteLine("Press any key to perform a request...");
    Console.ReadKey();

    Console.WriteLine("Cluster node: " + client.GetCluster().GetNode());
}

class MyRetryPolicy : IClientRetryPolicy
{
    public bool ShouldRetry(IClientRetryPolicyContext context)
    {
        Console.WriteLine($"Operation {context.Operation} has failed with error '{context.Exception.Message}'.");
        return true;
    }
}
```

* Start Ignite server in Docker with `docker run -p 10800:10800 apacheignite/ignite`.
* Run the code above.
* While it is waiting for input, stop the server and start it again. Connection loss won't be detected immediately, because heartbeats are not enabled in this example.
* Press a key in the program console. Operation failure will be logged, then the connection will be restored and the API call will be successfully retried.

We've used a custom `IClientRetryPolicy` implementation here to see how it works. Predefined implementations are available: [ClientRetryReadPolicy](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.ClientRetryReadPolicy.html),
[ClientRetryAllPolicy](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.ClientRetryAllPolicy.html).

**WARNING:** keep in mind that non-idempotent operations are not safe to retry. For example, an SQL query that increments a value might end up being executed twice because a connection has failed during the response phase.
`ClientRetryReadPolicy` only retries read operations and is safe to use.

More details:

* [IEP-82 Thin Client Retry Policy](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=195727946)
* [IgniteClientConfiguration.RetryPolicy](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.IgniteClientConfiguration.html#Apache_Ignite_Core_Client_IgniteClientConfiguration_RetryPolicy)

# Thin Client Service Descriptors

Services API is available in .NET thin client since [Ignite 2.10](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.10/), but `GetServiceDescriptors` method was missing.
It is now available:

```csharp
foreach (var desc in client.GetServices().GetServiceDescriptors())
{
    Console.WriteLine($"Service '{desc.Name}' is written in {desc.PlatformType}.");
}
```

# SendServerExceptionStackTraceToClient

When [Compute](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.Compute.IComputeClient.html) or [Service](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Client.Services.IServicesClient.html)
call fails due to a server-side exception, only the message is sent back to the client. We may have to dig through the server logs to understand the root cause of the issue.

To improve the diagnostics experience during development, server-side exception stack traces can be optionally sent to the client:

```csharp
var serverCfg = new IgniteConfiguration
{
    ClientConnectorConfiguration = new ClientConnectorConfiguration
    {
        ThinClientConfiguration = new ThinClientConfiguration
        {
            SendServerExceptionStackTraceToClient = true
        }
    }
};

using var server = Ignition.Start(serverCfg);
using var client = Ignition.StartClient(new IgniteClientConfiguration("127.0.0.1"));

client.GetServices().GetServiceProxy<IService>("foo-bar");
```

This option is disabled by default for security reasons: [stack traces contain information that can potentially aid an attacker](https://cwe.mitre.org/data/definitions/497.html).


# Links

* Main blog post: [https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0](https://blogs.apache.org/ignite/entry/apache-ignite-2-13-0)
* Full release notes: [https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt](https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt)
* Download: [https://ignite.apache.org/download](https://ignite.apache.org/download.cgi)
