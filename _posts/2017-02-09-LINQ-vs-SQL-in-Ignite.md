---
layout: post
title: LINQ vs SQL in Ignite.NET
---

LINQ has many benefits over SQL, but at what cost?

# Benchmark Results

Let's get straight to the results!

// TODO: Add string array conversion to demostrate compiled advantage

```
            Method |      Median |    StdDev |
------------------ |------------ |---------- |
         QueryLinq | 175.8261 us | 9.9202 us |
          QuerySql |  62.2791 us | 5.4908 us |
 QueryLinqCompiled |  57.9274 us | 3.1307 us |
```

Code is at [github.com/ptupitsyn/IgniteNetBenchmarks](https://github.com/ptupitsyn/IgniteNetBenchmarks/blob/master/IgniteLinqBenchmark.cs).

This is a comparison of equivalent queries via SQL, LINQ and Compiled LINQ.
Query is very simple (`select Name from SqlPerson where (SqlPerson.Id < ?)`), data set is very small (40 items, 20 returned): this exposes LINQ overhead better.

We can see right away that LINQ is a lot slower than raw SQL, but compiled LINQ is a bit faster bit faster.
Note that results are in *micro*seconds: real-world queries may take tens or even hundreds of *milli*seconds, so LINQ overhead will be hardly noticeable.

Anyway, how can we explain these results? Why compiled LINQ is faster than raw SQL?

# How Ignite LINQ Works

```cs
ICache<int, SqlPerson> cache = ignite.GetCache<int, SqlPerson>("persons");

IQueryable<int> qry = cache.AsCacheQueryable().Select(x => x.Value.Age);

int[] res = qry.ToArray();
```

If we run the above code in Visual Studio debugger and look at `qry` variable, we'll see something like this:

![ICacheQueryable Debug View](../images/Linq-vs-Sql/ICacheQueryable-debug.png)

Compiler has translated `.Select(x => x.Value.Age)` to an Expression Tree and passed it to `CacheFieldsQueryProvider`,
which, as we can see, turns into a regular Ignite.NET `SqlFieldsQuery`. This process is not free, that's where the overhead comes from.

We can get that `SqlFieldsQuery` and run it manually:

```cs
SqlFieldsQuery fieldsQry = ((ICacheQueryable)cache.AsCacheQueryable().Select(x => x.Value.Age)).GetFieldsQuery();

IList<IList> res = cache.QueryFields(fieldsQry).GetAll();
```

However, LINQ produces `int[]`, and fields query always returns a list, where each element is a list with a single int element. How is this achieved?
You may think that LINQ engine iterates over `IList` returned from `QueryFields` and populates an `int` array, but it is more clever than that.