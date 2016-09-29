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
            Id = int.MinValue,              // 4 bytes
            Name = "John Johnson",          // 12 bytes
            Data = new string('g', 1000),   // 1000 bytes
            Guid = Guid.NewGuid()           // 16 bytes
        };
    }
}
```

Full source code is available on [GitHub](https://github.com/ptupitsyn/IgniteNetBenchmarks).

Resulting size in serialized form:

              Method | Size (bytes) |
-------------------- | ------------ |
        Serializable |  1426        |
          Reflective |  1076        |
         Binarizable |  1076        |
      Reflective Raw |  1067        |
     Binarizable Raw |  1067        |
            Protobuf |  1048        |
            Payload  |  1032        |

Last row (Payload) is the raw data size (without string lengths even).

* As expected, Reflective and Binarizable produce exactly the same result (reflective does the same thing as we do in IBinarizable implementation).
* Raw mode is 9 bytes shorter
* Protobuf is 19 bytes shorter than Ignite raw
* Serializable takes quite a lot more space

To understand the differences, let's see how `int` field is written:

* Ignite non-raw: 1 byte type code, 4 bytes value
* Ignite raw: 4 bytes value
* Protobuf: 1 byte field key (wire type + field number), 1-5 bytes value

Protobuf is simply a set of fieldKey+value pairs (see [docs](https://developers.google.com/protocol-buffers/docs/encoding)), while Ignite serialized object always has a header of 24 bytes (which contains, among other things, protocol version and type information).
Protobuf format is quite similar to Ignite non-raw format, and, as we can see, the size is quite close (Ignite non-raw without header is 1052 bytes).

# Speed Comparison

[BenchmarkDotNet](https://github.com/PerfDotNet/BenchmarkDotNet) from Andrey Akinshin is used to measure performance.

Test methods serialize the data object to a byte array, deserialize it back, and verify equality. Default benchmark settings are used.

Ignite serializer can not be called directly, so we have to use Reflection to access the [Marshaller class](https://github.com/apache/ignite/blob/master/modules/platforms/dotnet/Apache.Ignite.Core/Impl/Binary/Marshaller.cs).

There are multiple classes that derive from `Person` class in order to configure different modes in Ignite.

Results:

              Method |     Median |    StdDev |
-------------------- |----------- |---------- |
        Serializable | 54.7112 us | 3.2140 us |
            Protobuf |  8.6584 us | 0.1308 us |
          Reflective |  8.4182 us | 0.1102 us |
         Binarizable |  7.9148 us | 0.1148 us |
      Reflective Raw |  7.8528 us | 0.1163 us |
     Binarizable Raw |  7.4358 us | 0.1583 us |

Summary:

* Ignite is on par with Protobuf on this particular data set (keep in mind that this can vary quite a bit depending on field types and values).
* Ignite raw mode is ~7% faster than non-raw mode. The gap will be wider when there are more fields.
* Reflective serialization is ~5% slower than manual Binarizable implementation (Ignite compiles a list of write & read method calls, so there is some overhead due to list iteration and delegate invocation).
* Serializable is a lot slower. The only advantage here is that we don't need to register the type in BinaryConfiguration.

As a conclusion, we can say that non-raw reflective Ignite serialization is perfectly fine for most use cases.
It enables all Ignite features (SQL, binary mode, etc), does not require user code modification, is fast and compact.