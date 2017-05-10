---
layout: post
title: What's new in Apache Ignite.NET 2.0
---

Apache Ignite 2.0 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-0-redesigned) last week.
Changes on Java side are tremendous, but Ignite.NET has some cool things to offer as well.

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
    // Zero configuration.
    var ignite = Ignition.Start();
    
    // Put arbitrary data to cache.
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

As you can see, the possibilities are endless!


# Plugin System

Ignite.NET is built on top of Ignite Java: .NET code starts a JVM in process and communicates with it via JNI and unmanaged memory regions, so that code is reused as much as possible, and performance is maximized.

Ignite 2.0 exposes this interoperation subsystem in form of plugins: each plugin consists of .NET and Java parts which can establish two-way communication and expose new APIs to the users.

In the next blog post we will implement such a plugin; for now you can have a look at the docs: [apacheignite-net.readme.io/docs/plugins](https://apacheignite-net.readme.io/docs/plugins).

# ASP.NET

A number of improvements has been made to address the issue with IIS application lifecycle, where AppDomains are loaded and unloaded within the same worker process.
Java part of Ignite does not belong to an `AppDomain`, so additional steps were required to deal with it.

In Ignite.NET 2.0:

1. `Ignition.StopAll(true)` is called automatically on domain unload, so that Java part is cleaned up properly.
2. `IgniteConfiguration.AutoGenerateIgniteInstanceName` property added to avoid instance name (`GridName` earlier) conflicts.
3. `Ignition.GetIgnite()` returns Ignite instance with any name, as long as there is only one instance present.

As a result, Ignite.NET can be easily used in IIS environment when `AutoGenerateIgniteInstanceName` is `true`:

* Configure everything in `web.config`:

```xml
<configuration>
    <configSections>
        <section name="igniteConfiguration" type="Apache.Ignite.Core.IgniteConfigurationSection, Apache.Ignite.Core" />
    </configSections>

    <igniteConfiguration autoGenerateIgniteInstanceName="true">
        <cacheConfiguration>
            <cacheConfiguration name='myWebCache' />
        </cacheConfiguration>
    </igniteConfiguration>
</configuration>
```

* Call `Ignition.GetIgnite()` whenever you need to work with Ignite in code.


# LINQ

As usual, Ignite LINQ to SQL engine continues to improve.

This time changes are subtle, but still important: all limitations to splitting queries into multiple statements or joining multiple subqueries into one has been addressed, so all of the following queries work:

```cs
// Join with separate variables:
var contracts = GetCache<Contract>().AsCacheQueryable();
var paymentPlans = GetCache<PaymentPlan>().AsCacheQueryable();
var query = from ic in contracts
    from pp in paymentPlans
    select pp.Value;

// One statement join:
var query = from ic in GetCache<Contract>().AsCacheQueryable()
    from pp in GetCache<PaymentPlan>().AsCacheQueryable()
    select pp.Value;

// Contains with separate variable:
var orgIds = orgsQry.Where(o => o.Value.Size < 100000).Select(o => o.Key);
		
var res = personsQry.Where(x => orgIds.Contains(x.Value.OrgId));

// One statement:
var res = personsQry.Where(x => orgsQry.Where(o => o.Value.Size < 100000)
                                .Select(o => o.Key).Contains(x.Value.OrgId));
```

# Other Improvements

Full release notes are there: [ignite.apache.org/releases/2.0.0/release_notes.html](https://ignite.apache.org/releases/2.0.0/release_notes.html).

Among others:

* Types are now mapped using `Type.FullName` instead of `Type.Name` during serialization, which reduces the chance of conflict. However, this [affects Java interoperability](https://apacheignite-net.readme.io/docs/platform-interoperability).
* `ICacheStore` is now generic. This provides a cleaner API and removes boxing/unboxing overhead.
* Java and .NET nodes can join with default configs, no extra steps required.
* F# record types serialization is fixed, so that SQL works as expected (I have plans for 'Ignite.NET in F#' blog post).


# Breaking Changes

There are some API changes that may break compilation and/or behavior in existing code, please refer to [Migration Guide](https://cwiki.apache.org/confluence/display/IGNITE/Apache+Ignite+2.0+Migration+Guide#ApacheIgnite2.0MigrationGuide-Ignite.NET).