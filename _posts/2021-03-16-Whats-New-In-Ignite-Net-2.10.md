---
layout: post
title: What's new in Apache Ignite.NET 2.10
---

[Apache Ignite](https://ignite.apache.org/) 2.10 has been released. Let's see what's new on .NET side of things.  


# Thin Client Services

.NET thin client can now invoke Ignite Services.
The service can be implemented in any language and should be deployed from server or thick client.

### Deploy .NET Service

```cs
var ignite = Ignition.Start();
ignite.GetServices().DeployClusterSingleton("Greeting", new GreetingService());

...

class GreetingService : IService
{
    // Empty Init/Execute/Cancel implementations omitted
    public string GetGreeting(string name) => 
        $"Hello {name} from {RuntimeInformation.FrameworkDescription}!";
}
``` 


### Deploy Java Service

```java
Ignite ignite = Ignition.start();
ignite.services().deployClusterSingleton("Greeting", new GreetingService());

...

class GreetingService implements Service
{
    // Empty init/execute/cancel implementations omitted
    @PlatformServiceMethod("GetGreeting")
    public String getGreeting(String name) {
        return "Hello " + name + " from Java!";
    }
}
```


### Invoke from .NET Thin Client

```cs
var client = Ignition.StartClient(new IgniteClientConfiguration("127.0.0.1"));

var result = client.GetServices()
    .GetServiceProxy<IGreetingService>("Greeting")
    .GetGreeting("John");

Console.WriteLine(result);

public interface IGreetingService
{
    string GetGreeting(string name);
}
```

Notice that:
* Invocation does not share any code with the implementation.
* Java and .NET services are invoked the same way - client side does not need to know about implementation details, 
  and we can even rewrite the service in a different language without the clients noticing.

# Thin Client Transactions

Transactions are now supported in the thin client, both explicit and ambient:

```cs
var client = Ignition.StartClient(new IgniteClientConfiguration("127.0.0.1"));

var cache = client.GetOrCreateCache<int, int>(new CacheClientConfiguration
{
    Name = "tx",
    AtomicityMode = CacheAtomicityMode.Transactional
});

// Ambient.
using (var ts = new TransactionScope())
{
    cache[1] = 1;
    cache[2] = 2;
    ts.Complete();
}

// Explicit.
using (var tx = client.GetTransactions().TxStart())
{
    cache[1] = 11;
    cache[2] = 22;
    tx.Commit();
}
```

Make sure to set `CacheAtomicityMode.Transactional` - default is `Atomic`, where transactions are ignored.


# ContinuousQuery.IncludeExpired

Continuous queries can include Expired events now. It has to be explicitly enabled:

```cs
cache.QueryContinuous(new ContinuousQueryClient<int, int>
{
    Listener = new Listener(),
    IncludeExpired = true
});

cache.WithExpiryPolicy(new ExpiryPolicy(TimeSpan.FromSeconds(1), null, null)).Put(10, 20);
Thread.Sleep(2000);

...

public class Listener : ICacheEntryEventListener<int, int>
{
    public void OnEvent(IEnumerable<ICacheEntryEvent<int, int>> evts)
    {
        foreach (var e in evts)
        {
            Console.WriteLine($"{e.EventType}: {e.Key}");
        }
    }
}
```

And the output is:
```
Created: 10
Expired: 10
```

This flag is available in both APIs - thin and thick.


# CacheConfiguration.NodeFilter

By default, cache data is distributed across all cluster nodes, according to the [affinity function](https://ignite.apache.org/docs/latest/data-modeling/data-partitioning).

`CacheConfiguration.NodeFilter` can restrict the node set for a given cache based on a node attribute.

```cs
TODO
```


This can be useful when different node groups have different responsibilities.
For example, we have a service deployed to a subset of nodes, and we want the data for that service to be located on the same subset of nodes.


# Wrap-up
 
Please see [release notes](https://ignite.apache.org/releases/2.10.0/release_notes.html) for a full list of new features, fixes, and improvements.

TODO: Links to release notes and other blogs
TODO: What's coming up in Ignite 2.11? 
