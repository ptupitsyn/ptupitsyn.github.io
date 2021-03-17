---
layout: post
title: What's new in Apache Ignite.NET 2.10
---

[Apache Ignite](https://ignite.apache.org/) 2.10 has been released. Let's see what's new on .NET side of things.  


# Thin Client Services

Thin client can now invoke Ignite Services. The service can be implemented in any language and should be deployed from server or thick client.

### Deploy .NET Service

```cs
var ignite = Ignition.Start();
ignite.GetServices().DeployClusterSingleton("Greeting.NET", new GreetingService());

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
ignite.services().deployClusterSingleton("Greeting.Java", new GreetingService());

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


# Thin Client Transactions


# ContinuousQuery.IncludeExpired



# Other Improvements

* `CacheConfiguration.NodeFilter`
* `SqlFieldsQuery.Partitions` and `SqlFieldsQuery.UpdateBatchSize` 
* `RendezvousAffinityFunction.BackupFilter`


# Wrap-up
 
Please see [release notes](https://ignite.apache.org/releases/2.10.0/release_notes.html) for a full list of new features, fixes, and improvements.

TODO: Links to release notes and other blogs
TODO: What's coming up in Ignite 2.11? 
