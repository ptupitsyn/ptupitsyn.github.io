---
layout: post
title: Implementing Ignite.NET Plugin: Distributed Semaphore
---

[Ignite.NET 2.0](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.0/) introduced [plugin system](https://apacheignite-net.readme.io/docs/plugins). Plugins can be .NET-only or .NET + Java. Let's see how to implement the latter.


# Why would I need a plugin?

Ignite.NET is built on top of Ignite (which is written in Java). JVM is started within .NET process, and .NET part talks to Java part and reuses existing Ignite functionality where possible.

Plugin system exposes this platform interaction mechanism to the third parties.
One of the main use cases is making Ignite and third party Java APIs available in .NET.

Good example of such an API is [IgniteSemaphore](https://apacheignite.readme.io/docs/distributed-semaphore), which is not yet available in Ignite.NET.


# Distributed Semaphore API

Ignite Semaphore is very similar to `System.Threading.Semaphore` ([MSDN](https://msdn.microsoft.com/en-us/library/system.threading.semaphore.aspx)), but the effect is cluster-wide: limit the number of threads executing a giving piece of code *across all Ignite nodes*.

It should be used in C# code like this:

```cs
IIgnite ignite = Ignition.GetIgnite();
ISemaphore semaphore = ignite.GetOrCreateSemaphore(name: "foo", count: 3);

semaphore.WaitOne();  // Enter the semaphore (may block)
// Do work
semaphore.Release();
```

Looks simple enough and quite useful; same API as built-in .NET `Semaphore`. Obviously, we can't change `IIgnite` interface, so `GetOrCreateSemaphore` is an extension method. Now onto the implementation!


# Java Plugin

Let's start with Java side of things. We need a way to call `Ignite.semaphore()` method there and provide access to the resulting instance to the .NET platform.

Create a Java project and reference Ignite 2.0 from Maven (detailed instructions can be found in [Building Multi-Platform Ignite Cluster](https://ptupitsyn.github.io/Ignite-Multi-Platform-Cluster/) post).

Every plugin starts with `PluginConfiguration`. Our plugin does not need any configuration properties, but the class must exist, so just make a simple one:

```java
public class IgniteNetSemaphorePluginConfiguration implements PluginConfiguration {}
```

Then comes the entry point: `PluginProvider<PluginConfiguration>`. This interface has lots of methods, but most of them can be left empty 
(`name` and `version` must not be null, so put something in there).
We are interested only in `initExtensions` method, which allows us to provide a cross-platform interoperation entry point. This is done by registering `PlatformPluginExtension` implementation:

```java
public class IgniteNetSemaphorePluginProvider implements PluginProvider<IgniteNetSemaphorePluginConfiguration> {
    public String name() {
        return "DotNetSemaphore";
    }

    public String version() {
        return "1.0";
    }

    public void initExtensions(PluginContext pluginContext, ExtensionRegistry extensionRegistry) 
            throws IgniteCheckedException {
        extensionRegistry.registerExtension(PlatformPluginExtension.class,
                new IgniteNetSemaphorePluginExtension(pluginContext.grid()));
    }

...

}
```