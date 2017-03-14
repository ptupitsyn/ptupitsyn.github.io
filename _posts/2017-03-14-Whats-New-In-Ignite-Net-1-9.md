---
layout: post
title: What's new in Apache Ignite.NET 1.9
---

Apache Ignite 1.9 [has been released](https://ignite.apache.org/news.html#apache-ignite-1.9-release) last week. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)

# TransactionScope API

Before 1.9, Ignite transactions could only be started explicitly with `IIgnite.GetTransactions().TxStart()` call.

Now you can also use standard `TransactionScope` API:

```cs
using (var ts = new TransactionScope())
{
  cache[1] = 2;
  cache[2] = 1;

  // Transaction has been started on first transactional ICache method call.
  Debug.Assert(ignite.GetTransactions().Tx != null);

  ts.Complete();
}
```

Besides being easier to use, `TransactionScope` API allows performing transactional operations across multiple systems (via [Two Phase Commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)). For example, you can update SQL Server database and Ignite cache within a single `TransactionScope` block. All operations will participate in a common transaction, so that either both systems are updated, or all changes are reverted.

Documentation page: [apacheignite-net.readme.io/docs/transactionscope-api](https://apacheignite-net.readme.io/docs/transactionscope-api)

# Distributed DML

In previous versions, Ignite SQL worked only in data retrieval mode (`SELECT` statements). [Data Manipulation Language](https://en.wikipedia.org/wiki/Data_manipulation_language) statements (`INSERT`, `UPDATE`, `DELETE`, `MERGE`) support has been added in 1.9 via existing `QueryFields` API:

```cs
// Insert person as object:
cache.QueryFields(new SqlFieldsQuery("INSERT INTO Person(_key, _val) VALUES(?, ?)", 1L, new Person("John", "Smith")));

// Insert person as fields:
cache.QueryFields(new SqlFieldsQuery(
    "INSERT INTO Person(_key, firstName, lastName) VALUES(?, ?, ?)", 1L, "John", "Smith"));
```

These statements are translated to `ICache.Put()` and `ICache.InvokeAll()` calls.

Documentation page: [apacheignite-net.readme.io/docs/distributed-dml](https://apacheignite-net.readme.io/docs/distributed-dml)

# LINQ Improvements

Distributed LINQ continues to evolve:

## Contains

`IEnumerable.Contains` extension method is now supported:

```cs
var persons = ignite.GetCache<int, Person>("persons").AsCacheQueryable();

// Select persons with specific ids using inline collection:
var res = cache.Select(x => x.Value).Where(p => new[] {1, 2, 3}.Contains(p.Id));

// Generated SQL:
// select _T0._val from "persons".Person as _T0 where (_T0.Id IN (?, ?, ?))
```

Keep in mind that queries with `IN` clause are not always optimal: [apacheignite-net.readme.io/docs/sql-queries#section-performance-and-usability-considerations](https://apacheignite-net.readme.io/docs/sql-queries#section-performance-and-usability-considerations).

## DateTime Properties

The following `DateTime` properties can be used in LINQ: `Year`, `Month`, `Day`, `Hour`, `Minute`, `Second`, `DayOfYear`, `DayOfWeek`. 

```cs

```

## Inline Joins
