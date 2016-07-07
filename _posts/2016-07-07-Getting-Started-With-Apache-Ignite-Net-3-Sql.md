---
layout: post
title: Getting Started with Apache Ignite.NET Part 3&#58; Cache Queries
---

This part covers cache queries: Scan, SQL, LINQ, and Text.

* [Part 1: Getting Started](https://ptupitsyn.github.io/Getting-Started-With-Apache-Ignite-Net/)
* [Part 2: Distributed Cache](https://ptupitsyn.github.io/Getting-Started-With-Apache-Ignite-Net-2-Cache/)
* Part 3: Cache Queries

## Query Types

Cache queries provide a way to retrieve multiple cache entries based on a condition. Since `ICache<K, V>` implements `IEnumerable<ICacheEntry<K, V>>`, the simplest way to find some entries is to enumerate the cache (via `foreach` or LINQ to objects). 

However, in a distributed system this will cause all cache entries to be transmitted to a local machine and filtered there:
 * Irrelevant entries get transmitted over the network
 * Only one node of the distributed system does the filtering


 More efficient approach is to filter the entries *before* sending them to the requesting node, minimizing network overhead and splitting the filtering load between the nodes. Ignite has multiple query mechanisms that achieve this:
 * `ScanQuery`: user-defined filter predicate is sent to remote nodes, executed, and matching entries are sent back.
 * `SqlQuery` (and `SqlFieldsQuery`): write SQL as you will do normally, and Ignite will take care of executing the query on multiple nodes and aggregating the result.
 * `TextQuery`: [Lucene](https://lucene.apache.org/core/)-based full-text search. Similarly to SQL, you write a query in [Lucene syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html), and Ignite takes care of executing it in a distributed manner.  

## Scan Queries
To demonstrate the query API, we'll use our Person class from the previous post: 

```cs
static void Main()
{
    var cfg = new IgniteConfiguration
    {
        BinaryConfiguration = new BinaryConfiguration(typeof(Person), typeof(PersonFilter))
    };
    IIgnite ignite = Ignition.Start(cfg);
    
    ICache<int, Person> cache = ignite.GetOrCreateCache<int, Person>("persons");
    cache[1] = new Person {Name = "John Doe", Age = 27};
    cache[2] = new Person { Name = "Jane Moe", Age = 43 };

    var scanQuery = new ScanQuery<int, Person>(new PersonFilter());
    IQueryCursor<ICacheEntry<int, Person>> queryCursor = cache.Query(scanQuery);

    foreach (ICacheEntry<int, Person> cacheEntry in queryCursor)
        Console.WriteLine(cacheEntry);
}

class Person
{
    public string Name { get; set; }
    public int Age { get; set; }

    public override string ToString()
    {
        return $"Person [Name={Name}, Age={Age}]";
    }
}

class PersonFilter : ICacheEntryFilter<int, Person>
{
    public bool Invoke(ICacheEntry<int, Person> entry)
    {
        return entry.Value.Age > 30;
    }
}

```

And the output is `CacheEntry [Key=2, Value=Person [Name=Jane Moe, Age=43]]`.

Notice that we had to register `PersonFilter` in the `BinaryConfiguration`, because this filter is sent over the wire to remote nodes and needs to be serialized.

Scan queries are the simplest form of distributed queries, do not require any cache configuration changes, and allow any user-defined logic in the filter. However, they are limited to simple data filtering; there are no projections, joins, or aggregates.

## SQL Queries

SQL Queries require explicit configuration. We need to tell which types and fields should be available in SQL queries.

First, let's decorate `Person` class properties with `[QuerySqlField]` to add them to the SQL engine:

```cs
class Person
{
    [QuerySqlField]
    public string Name { get; set; }
    
    [QuerySqlField]
    public int Age { get; set; }
    
    ...
}

```  

After that we tell Ignite to enable SQL queries for Person class via `CacheConfiguration`:

```cs
ICache<int, Person> cache = ignite.GetOrCreateCache<int, Person>(
    new CacheConfiguration("persons", typeof(Person)));
```

And now we can replace the `ScanQuery` with equivalent `SqlQuery`:

```cs
var sqlQuery = new SqlQuery(typeof(Person), "WHERE age > ?", 30);
IQueryCursor<ICacheEntry<int, Person>> queryCursor = cache.Query(sqlQuery);
```

And the output will be the same: `CacheEntry [Key=2, Value=Person [Name=Jane Moe, Age=43]]`. 

Queries are parametrized with `?` symbol, and parameter values are provided in the same order after the query text. `SELECT` and `FROM` can be omitted since there is only one SQL-enabled type in the cache, and we don't select individual fields.

Notice that we still use a single NuGet package, our program fits on screen, and we run a full-fledged SQL over our custom data. *How cool is that?*




---
// TODO:
How SQL queries work; H2 Debug Console