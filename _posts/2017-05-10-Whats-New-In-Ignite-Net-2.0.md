---
layout: post
title: What's new in Apache Ignite.NET 2.0
---

Apache Ignite 2.0 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-0-redesigned) last week. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)


# Dynamic Type Registration

This is a huge one! In short:

* No need for `BinaryConfiguration` anymore. Types are registered within cluster automatically.
* Everything is serialized in Ignite format, no more limitations for `[Serializable]` and `ISerializable`, SQL always works.
* **Anything** can be serialized! Classes, structs, system types, delegates, anonymous types, expression trees, generics, you name it!

Now let's see this in action.
First, you **don't need to register types** in `BinaryConfiguration` to be able to use them in Ignite, so the following works:

```cs
// Just a simple class, no attributes or interfaces.
class Person
{
    public string Name { get; set; }
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
    // Keep in mind that assembly with ComputeAction must be present on all nodes.
    ignite.GetCompute().Broadcast(new ComputeAction());
}
```

Next, `[Serializable]` and `ISerializable` types are written in Ignite binary format, which means **SQL works for `[Serializable]` and `ISerializable`**:

```cs
class Person : ISerializable
{
    [QuerySqlField]
    public string Name { get; set; }

    public Person() { }

    public Person(SerializationInfo info, StreamingContext context) 
        => Name = info.GetString("Name");

    public void GetObjectData(SerializationInfo info, StreamingContext context) 
        => info.AddValue("Name", Name);
}

void Main()
{
    var ignite = Ignition.Start();

    var cache = ignite.CreateCache<int, Person>(
        new CacheConfiguration("persons", typeof(Person)));
    
    cache[1] = new Person { Name = "John Doe" };

    var res = cache.QueryFields(new SqlFieldsQuery(
        "select name from person where name like 'John%'")).GetAll();

    Console.WriteLine(res[0][0]);  // John Doe
}
```

`ISerializable` classes are serialized the same way as `IBinarizable`, the difference is only in API.
See [apacheignite-net.readme.io/docs/serialization](https://apacheignite-net.readme.io/docs/serialization) for more details.

Finally, as a consequence of the above, **anything and everything can be serialized in Ignite**, including object hierarchies that combine `ISerializable` and regular types.
I think the most awesome example here are **Expression Trees**. There is no built-in way in .NET to serialize an
Expression Tree (see [google search for 'serialize expression tree'](https://www.google.ru/search?q=serialize+expression+tree)), but you can do this in Ignite no problemo:

```cs
using (var ignite = Ignition.Start())
{
    var cache = ignite.CreateCache<int, Expression<Func<int, int>>>("c");

    Expression<Func<int, int>> addOne = x => x + 1;

    cache[1] = addOne;

    var result = cache[1];

    Console.WriteLine(result.Compile().Invoke(5));  // 6
}
```

We have put a piece of code into distributed cache, and any node can easily retrieve and execute it!

### More Tricks

Below are some fancy tricks you can do with dynamic serializer:

```cs
using (var ignite = Ignition.Start())
{
    var cache = ignite.CreateCache<int, object>("c");

    // Type instance.
    cache[1] = typeof(int);
    Console.WriteLine(ReferenceEquals(typeof(int), cache[1]));  // true

    // Delegate.
    var greeting = "Hi!";
    cache[2] = (Action) (() => { Console.WriteLine(greeting); });
    ((Action) cache[2])();  // Hi!

    // Modify serialized delegate.
    var binDelegate = binCache[2];
    var target = binDelegate.GetField<IBinaryObject>("target0");
    target = target.ToBuilder().SetField("greeting", "Woot!").Build();
    binCache[2] = binDelegate.ToBuilder().SetField("target0", target).Build();

    ((Action)cache[2])();  // Woot!

    // Anonymous type.
    cache[3] = new { Foo = "foo", Bar = 42 };
    Console.WriteLine(binCache[3]);  // ...[<Bar>i__Field=42, <Foo>i__Field=foo]

    // Dynamic ExpandoObject.
    dynamic dynObj = new ExpandoObject();
    dynObj.Baz = "baz";
    dynObj.Qux = 1.28;

    cache[4] = dynObj;
    Console.WriteLine(binCache[4]); // _keys=[Baz, Qux], _dataArray=[baz, 1.28, ]
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