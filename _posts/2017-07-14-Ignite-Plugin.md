---
layout: post
title: Implementing Ignite.NET Plugin&#58; Distributed Semaphore
---

[Ignite.NET 2.0](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.0/) introduced [plugin system](https://apacheignite-net.readme.io/docs/plugins). Plugins can be .NET-only or .NET + Java. Let's see how to implement the latter.


# Why would I need a plugin?

Ignite.NET is built on top of Ignite (which is written in Java). JVM is started within .NET process, and .NET part talks to Java part and reuses existing Ignite functionality where possible.

Plugin system exposes this platform interaction mechanism to the third parties.
One of the main use cases is making Ignite and third party Java APIs available in .NET.

Good example of such an API is [IgniteSemaphore](https://apacheignite.readme.io/docs/distributed-semaphore), which is not yet available in Ignite.NET.

All source code for this post is available on GitHub: [github.com/ptupitsyn/ignite-net-examples/tree/master/Plugin](https://github.com/ptupitsyn/ignite-net-examples/tree/master/Plugin).


# Distributed Semaphore API

Ignite Semaphore is very similar to `System.Threading.Semaphore` ([MSDN](https://msdn.microsoft.com/en-us/library/system.threading.semaphore.aspx)), but the effect is cluster-wide: limit the number of threads executing a given piece of code *across all Ignite nodes*.

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

Then comes the plugin entry point: `PluginProvider<PluginConfiguration>`. This interface has lots of methods, but most of them can be left empty 
(`name` and `version` must not be null, so put something in there).
We are interested only in `initExtensions` method, which allows us to provide a cross-platform interoperation entry point. This is done by registering `PlatformPluginExtension` implementation:

```java
public class IgniteNetSemaphorePluginProvider implements PluginProvider<IgniteNetSemaphorePluginConfiguration> {
    public String name() { return "DotNetSemaphore"; }
    public String version() { return "1.0"; }

    public void initExtensions(PluginContext pluginContext, ExtensionRegistry extensionRegistry) 
            throws IgniteCheckedException {
        extensionRegistry.registerExtension(PlatformPluginExtension.class,
                new IgniteNetSemaphorePluginExtension(pluginContext.grid()));
    }
...
}
```

`PlatformPluginExtension` has a unique `id` to retrieve it from .NET side and a `PlatformTarget createTarget()` method to create an object that can be invoked from .NET.

[`PlatformTarget` interface in Java](https://github.com/apache/ignite/blob/master/modules/core/src/main/java/org/apache/ignite/internal/processors/platform/PlatformTarget.java)  mirrors [`IPlatformTarget` interface in .NET](https://github.com/apache/ignite/blob/master/modules/platforms/dotnet/Apache.Ignite.Core/Interop/IPlatformTarget.cs). When you call `IPlatformTarget.InLongOutLong` in .NET, `PlatformTarget.processInLongOutLong` is called in Java on your implementation. There are a number of other methods that allow exchanging primitives, serialized data, and objects. Each method has a `type` parameter which specifies an operation code, in case when there are many different methods on your plugin.

We are going to need two `PlatformTarget` classes: one that represents our plugin as a whole and has `getOrCreateSemaphore` method, and another one to represent each particular semaphore. First one should take a `string` name and `int` count and return an object, so we need to implement `PlatformTarget.processInStreamOutObject`. Other methods are not needed and can be left blank:

```java
public class IgniteNetPluginTarget implements PlatformTarget {
    private final Ignite ignite;

    public IgniteNetPluginTarget(Ignite ignite) {
        this.ignite = ignite;
    }

    public PlatformTarget processInStreamOutObject(int i, BinaryRawReaderEx binaryRawReaderEx) throws IgniteCheckedException {
        String name = binaryRawReaderEx.readString();
        int count = binaryRawReaderEx.readInt();

        IgniteSemaphore semaphore = ignite.semaphore(name, count, true, true);

        return new IgniteNetSemaphore(semaphore);
    }
...
}
```

For each `ISemaphore` object in .NET there will be one `IgniteNetSemaphore` in Java, which is also a `PlatformTarget`. And this object will handle `WaitOne` and `Release` methods and delegate them to underlying `IgniteSemaphore` object. Since both of these methods are void and parameterless, the simplest `PlatformTarget` method will work:

```java
public long processInLongOutLong(int i, long l) throws IgniteCheckedException {
    if (i == 0) semaphore.acquire();
    else semaphore.release();
    
    return 0;
}
```

That's it, Java part is implemented! We just need to make our `IgniteNetSemaphorePluginProvider` class available to Java service loader by creating a `resources\META-INF.services\org.apache.ignite.plugin.PluginProvider` file with a single line containing the class name. Package the project with Maven (`mvn package` in console, or use IDEA UI). There should be a `IgniteNetSemaphorePlugin-1.0-SNAPSHOT.jar` file in the `target` directory. We can move on to the .NET part now.


# .NET Plugin

First let's make sure our Java code gets picked up by Ignite. Create a console project, install Ignite NuGet package, and start Ignite with the path to the jar file that we just created:

```cs
var cfg = new IgniteConfiguration
{
    JvmClasspath = @"..\..\..\..\Java\target\IgniteNetSemaphorePlugin-1.0-SNAPSHOT.jar"
};

Ignition.Start(cfg);
```

Ignite node starts up and we should see our plugin name in the log:

```
[16:02:38] Configured plugins:
[16:02:38]   ^-- DotNetSemaphore 1.0
```

Great! For the .NET part we'll take an API-first approach: implement the extension method first and continue from there.

```cs
public static class IgniteExtensions
{
    public static Semaphore GetOrCreateSemaphore(this IIgnite ignite, string name, int count)
    {
        return ignite.GetPlugin<SemaphorePlugin>("semaphorePlugin").GetOrCreateSemaphore(name, count);
    }
}
```

For the `GetPlugin` method to work, `IgniteConfiguration.PluginConfigurations` property should be set. It takes a collection of `IPluginConfiguration` implementations, and each implementation must, in turn, link to a `IPluginProvider` implementation with an attribute:

```cs
[PluginProviderType(typeof(SemaphorePluginProvider))]
class SemaphorePluginConfiguration : IPluginConfiguration  {...}
```

On node startup Ignite.NET iterates through plugin configurations, instantiates plugin providers, and calls `Start(IPluginContext<SemaphorePluginConfiguration> context)` method on them. `IIgnite.GetPlugin` calls are then delegated to `IPluginProvider.GetPlugin` of the provider with specified name.

```cs
class SemaphorePluginProvider : IPluginProvider<SemaphorePluginConfiguration>
{
    private SemaphorePlugin _plugin;

    public T GetPlugin<T>() where T : class
    {
        return _plugin as T;
    }

    public void Start(IPluginContext<SemaphorePluginConfiguration> context)
    {
        _plugin = new SemaphorePlugin(context);
    }

    ...

}
```

`IPluginContext` provides access to Ignite instance, Ignite and plugin configurations, and has `GetExtension` method, which delegates to `PlatformPluginExtension.createTarget()` in Java. This way we "establish connection" between the two platforms. `IPlatformTarget` in .NET gets linked to `PlatformTarget` in Java; they can call each other, and the lifetime of Java object is tied to lifetime of .NET object. Once .NET object is reclaimed by the garbage collector, finalizer releases the Java object reference, and it will also be garbage collected.

Remaining implementation is simple - just call appropriate `IPlatformTarget` methods:

```cs
class SemaphorePlugin
{
    private readonly IPlatformTarget _target;  // Refers to IgniteNetPluginTarget in Java

    public SemaphorePlugin(IPluginContext<SemaphorePluginConfiguration> context)
    {
        _target = context.GetExtension(100);
    }

    public Semaphore GetOrCreateSemaphore(string name, int count)
    {
        var semaphoreTarget = _target.InStreamOutObject(0, w =>
        {
            w.WriteString(name);
            w.WriteInt(count);
        });

        return new Semaphore(semaphoreTarget);
    }
}

class Semaphore
{
    private readonly IPlatformTarget _target;  // Refers to IgniteNetSemaphore in Java

    public Semaphore(IPlatformTarget target)
    {
        _target = target;
    }

    public void WaitOne()
    {
        _target.InLongOutLong(0, 0);
    }

    public void Release()
    {
        _target.InLongOutLong(1, 0);
    }
}
```

We are done! Quite a bit of boilerplate code, but adding more logic to existing plugin is easy, just implement a pair of methods on both sides.
Ignite uses [JNI](https://en.wikipedia.org/wiki/Java_Native_Interface) and unmanaged memory to exchange data between .NET and Java platforms within single process, which is simple and efficient.

# Testing

To demonstrate distributed nature of our Semaphore, we can run multiple Ignite nodes where each of them calls `WaitOne()`. We'll see that only two nodes at a time are able to acquire the semaphore:

```cs
var ignite = Ignition.Start(cfg);
var sem = ignite.GetOrCreateSemaphore("foo", 2);

Console.WriteLine("Trying to acquire semaphore...");

sem.WaitOne();

Console.WriteLine("Semaphore acquired. Press any key to release.");
Console.ReadKey();
```

Download full project from GitHub: [github.com/ptupitsyn/ignite-net-examples/tree/master/Plugin](https://github.com/ptupitsyn/ignite-net-examples/tree/master/Plugin).