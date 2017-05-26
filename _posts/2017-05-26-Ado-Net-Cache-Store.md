---
layout: post
title: ADO.NET As Ignite.NET Cache Store
---

Implementing efficient [Ignite.NET](https://ignite.apache.org/) persistent store with ADO.NET and SQL Server: continue the story from [Entity Framework Cache Store](https://ptupitsyn.github.io/Entity-Framework-Cache-Store/).


# ADO.NET vs Entity Framework

Previous article, [Entity Framework Cache Store](https://ptupitsyn.github.io/Entity-Framework-Cache-Store/), describes a way to persist Ignite in-memory data in SQL server using Entity Framework. The code is nice and elegant, but not efficient (as mentioned on [reddit](https://www.reddit.com/r/programming/comments/593ep1/entity_framework_as_ignitenet_cache_store/d95mnt6/)), because converting object operations to SQL queries introduces overhead.

Today we are going to cut out all the middlemen:

* Work with SQL directly to read and write data
* Use binary mode in Ignite to avoid serialization costs

# Ignite 2.0 Generic Cache Store

Cache store interface has been reworked in [Ignite.NET 2.0](https://ptupitsyn.github.io/Whats-New-In-Ignite-Net-2.0/) to operate on generic arguments. This reduces casting and boxing, making code nicer and faster:

```cs
// Ignite 1.x
class MyStore : ICacheStore
{
    public object Load(object key) => db.Find((int) key);
}

// Ignite 2.x
class MyStore : ICacheStore<int, string>
{
    public string Load(int key) => db.Find(key);
}

```

# Ignite Binary Mode

By default, Ignite works with user-defined objects and types, serializing / deserializing them as needed. While this serialization is very [efficient](https://ptupitsyn.github.io/Ignite-Serialization-Performance/), it is still not free.

To squeeze every bit of performance there is a [binary mode](https://apacheignite-net.readme.io/docs/binary-mode) where we work with objects in serialized form, retrieving and modifying individual fields.

We are going to use this binary mode both on cache side and cache store side.

# Code

The full source code is on [github.com/ptupitsyn/ignite-net-examples](https://github.com/ptupitsyn/ignite-net-examples), under AdoNetCacheStore folder.

The project is self-sufficient, you can download the sources and run it without setting anything up.
It uses SQL Server Compact (via NuGet) and creates a database in the bin folder when needed.

# Data Model

Our model will be defined in SQL server like this:

```sql
CREATE TABLE Cars (ID int, Name NVARCHAR(200), Power int)
```

In Ignite this can be represented with `ICache<int, Car>` where `Car` class has `Name` and `Power` fields. However, we are going to use binary mode where classes are not needed:

```cs
// Retrieve cache and switch to binary mode.
ICache<int, IBinaryObject> cars = ignite.GetCache<int, object>("cars").WithKeepBinary<int, IBinaryObject>();

// Create new value with binary builder.
IBinaryObject car = ignite.GetBinary()
    .GetBuilder("Car")
    .SetStringField("Name", "Honda NSX")
    .SetIntField("Power", 600)
    .Build();

// Put to cache, this causes ICacheStore.Write call (when store is configured and write-through).
cars[1] = car;
```

# Cache Store Configuration

Configuration is almost the same as in Entity Framework store, but with important different: `KeepBinaryInStore` is `true`:

```cs
var cacheCfg = new CacheConfiguration
{
    Name = "cars",
    CacheStoreFactory = new AdoNetCacheStoreFactory(),
    KeepBinaryInStore = true,
    ReadThrough = true,
    WriteThrough = true
};
```

This way cache store implementation receives `IBinaryObject` instances directly without any deserialization.

# Implementing Cache Store

Let's look at `Write` method first, which is called under the hood of `cache.Put`:

```cs
public class AdoNetCacheStore : ICacheStore<int, IBinaryObject>
{
    // Notice that method arguments correspond to ICache<int, IBinaryObject> above.
    public void Write(int key, IBinaryObject val)
    {
        using (var conn = new SqlCeConnection(ConnectionString))
        {
            using (var cmd = new SqlCeCommand(@"INSERT INTO Cars (ID, name, Power) VALUES (@id, @name, @power)", conn))
            {
                cmd.Parameters.AddWithValue("@id", key);

                // Transfer data directly from binary object to SQL query.
                cmd.Parameters.AddWithValue("@name", val.GetField<string>("Name"));
                cmd.Parameters.AddWithValue("@power", val.GetField<int>("Power"));

                conn.Open();
                cmd.ExecuteNonQuery();
            }
        }
    }

    ...
}
```

There are no intermediate objects, we operate on raw field values here, which is as efficient as it gets.

Read method is in similar fashion:

```cs
public IBinaryObject Load(int key)
{
    using (var conn = new SqlCeConnection(ConnectionString))
    {
        using (var cmd = new SqlCeCommand(@"SELECT Name, Power FROM Cars WHERE Id = @id", conn))
        {
            cmd.Parameters.AddWithValue("@id", key);

            conn.Open();

            foreach (IDataRecord row in cmd.ExecuteReader())
            {
                // Return first record.
                return Ignite.GetBinary()
                    .GetBuilder("Car")
                    .SetStringField("Name", row.GetString(0))
                    .SetIntField("Power", row.GetInt32(1))
                    .Build();
            }

            return null;
        }
    }
}
```

Again, we create serialized object directly from the data reader, keeping allocations and overhead to a minimum.

# Running The Example

As you have probably noted, I use SQL Server Compact Edition (via NuGet). This way you don't need to install anything on your machine.
Just download the code (`git clone https://github.com/ptupitsyn/ignite-net-examples.git`), open `AdoNetCacheStore\AdoNetCacheStore.sln` and run.

Database will be created automatically. I recommend setting breakpoints on cache operations and cache store methods to see it all in action.

`ICacheStore.Delete` is not implemented, by the way. I leave it up to the readers to implement it and test by calling `cache.Remove(1)`.
