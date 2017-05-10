---
layout: post
title: What's new in Apache Ignite.NET 2.0
---

Apache Ignite 2.0 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-0-redesigned) last week. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)


# Dynamic Type Registration

This is a huge one! In short:

* No need for `BinaryConfiguration` anymore. Types are registered within cluster automatically.
* Everything is serialized in Ignite format, no more limitations for `[Serializable]`, SQL always works.
* **Anything** can be serialized! Classes, structs, system types, delegates, anonymous types, you name it!

Now let's see this in action.
First, you don't need to register types in `BinaryConfiguration` to be able to use them in Ignite, so the following works:

```cs
class Person
{
    string Name { get; set; }
}

class ComputeAction : IComputeAction
{
    public void Invoke() => Console.WriteLine("Hello World!");
}

void Main()
{
    var ignite = Ignition.Start();
    
    // Put arbitrary data to cache without prior configuration.
    var cache = ignite.CreateCache<int, Person>("persons");
    cache[1] = new Person { Name = "John Doe" };

    // Execute computations: print "Hello World!" on all nodes.
    ignite.GetCompute().Broadcast(new ComputeAction());
}

```

# Plugin System

TODO

# LINQ Improvements

TODO

# Other Improvements

TODO

# Breaking Changes

There are some API changes that may break compilation and/or behavior in existing code, please refer to [Migration Guide](https://cwiki.apache.org/confluence/display/IGNITE/Apache+Ignite+2.0+Migration+Guide#ApacheIgnite2.0MigrationGuide-Ignite.NET).