---
layout: post
title: LINQ vs SQL in Ignite.NET&#58; Performance
---

LINQ has many benefits over SQL, but at what cost?

# Benchmark Results

Let's get straight to the results!

```
            Method |      Median |    StdDev |
------------------ |------------ |---------- |
         QueryLinq | 175.8261 us | 9.9202 us |
          QuerySql |  62.2791 us | 5.4908 us |
 QueryLinqCompiled |  57.9274 us | 3.1307 us |
```

Code is at [github.com/ptupitsyn/IgniteNetBenchmarks](https://github.com/ptupitsyn/IgniteNetBenchmarks/blob/master/IgniteLinqBenchmark.cs).

This is a comparison of equivalent queries via SQL, LINQ and Compiled LINQ.
Query is very simple (`select Age from SqlPerson where (SqlPerson.Id < ?)`), data set is very small (40 items, 20 returned): this exposes LINQ overhead better.

We can see right away that LINQ is a lot slower than raw SQL, but compiled LINQ is a bit faster.
Note that results are in *micro*seconds: real-world queries may take tens or even hundreds of *milli*seconds, so LINQ overhead will be hardly noticeable.

Anyway, how can we explain these results? Why compiled LINQ is faster than raw SQL?

# How Ignite LINQ Works

```cs
ICache<int, SqlPerson> cache = ignite.GetCache<int, SqlPerson>("persons");

IQueryable<int> qry = cache.AsCacheQueryable().Select(x => x.Value.Age);

IList<int> res = qry.GetAll();
```

If we run the above code in Visual Studio debugger and look at `qry` variable, we'll see something like this:

![ICacheQueryable Debug View](../images/Linq-vs-Sql/ICacheQueryable-debug.png)

Compiler has translated `.Select(x => x.Value.Age)` to an Expression Tree and passed it to `CacheFieldsQueryProvider`,
which, as we can see, turns into a regular Ignite.NET `SqlFieldsQuery`. This process is not free, that's where the overhead comes from.

We can get that `SqlFieldsQuery` and run it manually:

```cs
SqlFieldsQuery fieldsQry = ((ICacheQueryable)cache.AsCacheQueryable().Select(x => x.Value.Age)).GetFieldsQuery();

IQueryable<IList> res = cache.QueryFields(fieldsQry);
```

However, LINQ produces typed `IQueryable<int>` instead of untyped `IQueryable<IList>`. How is this achieved?
You may think that LINQ engine iterates over `IQueryCursor` returned from `QueryFields` and populates `List<int>`, but it is more clever than that.

There is a hidden API, `ICacheInternal`, which has `IQueryCursor<T> QueryFields<T>(SqlFieldsQuery qry, Func<IBinaryRawReader, int, T> readerFunc)` method.
SQL engine returns fields query results as a raw memory stream where field values are written one after another.
So for a query above with one `int` field LINQ engine will produce the following code:

```cs
var cacheInt = (ICacheInternal) cache;
var fieldQry = new SqlFieldsQuery("SELECT Age from SqlPerson");
Func<IBinaryRawReader, int, int> readerFunc = (reader, fieldCount) => reader.ReadObject<int>();
IQueryCursor<int> cur = cacheInt.QueryFields(fieldQry, readerFunc);
```

This code produces zero extra allocations and zero type casts while reading query results. That is where LINQ advantage comes from: it is aware of resulting data types and can generate specialized deserialization code, while regular SQL query reads all field values as objects, which causes excessive allocations (`IList` for each row, boxing of value types) and requires type casting.

# Conclusion

LINQ is not only [much nicer to work with than SQL](https://www.linqpad.net/WhyLINQBeatsSQL.aspx), it can also be on par or faster when used properly! Just don't forget to use `CompiledQuery` when on a hot path.