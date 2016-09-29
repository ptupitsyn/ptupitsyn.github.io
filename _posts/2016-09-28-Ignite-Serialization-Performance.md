---
layout: post
title: Ignite.NET Serialization Performance
---

How fast are different Ignite serialization modes? How do they compare to other popular serializers?

# Reasoning

Ignite is a distributed system, and data serialization is at its core.
Cached data, compute closures, service instances, messages and events - everything is serialized to be transferred over the network.
Therefore, fast and compact serialization is essential for overall system performance.

Ignite uses own custom binary protocol, and there is a natural question: is it adequate? How does it compare to standard and third party serializers?

There are multiple modes available (see [Serialization](https://apacheignite-net.readme.io/docs/serialization) docs):

* Reflective: automatic opt-out serialization of all fields with metadata (random field access + SQL)
* Reflective Raw: same as above, but without metadata (sequential-only access, SQL is not available)
* Binarizable: manual interface implementation with field metadata
* Binarizable Raw: manual interface implementation without field metadata
* Serializable: standard BinaryFormatter (SQL is not available)

As for third party serializers, I'm going to use [protobuf-net](https://github.com/mgravell/protobuf-net), which is quite popular, quick, and compact.
There are a lot of "protobuf vs X" comparisons online, so you can compare Ignite to other serializers by comparing them to protobuf.

# Disclaimer

This post does not aim to provide very fair and comprehensive performance comparison of Ignite and Protobuf.

Ignite serializer is not a general purpose serializer (you can't even use it directly, as we'll see below), it is tailored specifically for distributed environment.
Our goal is to have a general idea of performance characteristics of various serialization modes.

# Size Comparison

The following data object is used for all comparisons, initialized by CreateInstance method:

```cs
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Data { get; set; }
    public Guid Guid { get; set; }

    public static T CreateInstance<T>() where T : Person, new()
    {
        return new T
        {
            Id = int.MinValue,
            Name = "John Johnson",
            Data = new string('g', 1000),
            Guid = Guid.NewGuid()
        };
    }
}
```

Full source code is available on [GitHub](https://github.com/ptupitsyn/IgniteNetBenchmarks).

Resulting size in serialized form:

              Method | Size (bytes) |
-------------------- |--------------|
        Serializable |  1426        |
          Reflective |  1076        |
         Binarizable |  1076        |
      Reflective Raw |  1067        |
     Binarizable Raw |  1067        |
            Protobuf |  1048        |


# Speed Comparison