---
layout: post
title: Dynamic LINQ with Ignite.NET and System.Linq.Dynamic.Core
---

TBD: Better title, more catchy? "Dynamic LINQ performance with Ignite.NET and System.Linq.Dynamic.Core"

![Apache Ignite Persistent Store](../images/ignite-dynamic-linq.png) Dynamically building database queries 
can be necessary for some use cases, such as UI-based filtering. 
This can get challenging with LINQ-based frameworks such as EF Core and Ignite.NET.

# Use Case Example

Let's say we are tasked with building a Web API like this:

```
GET /cars?make=Ford&model=Mustang&searchMode=Any
```

I'm going to use [Apache Ignite.NET](https://ignite.apache.org) as an example, but the same approach can be used with [EF Core](https://learn.microsoft.com/en-us/ef/core/).

The search mode can be `Any` or `All`, and it defines whether we should use `OR` or `AND` in the query. How do we build the query?
Let's start with retrieving `IQueryable` from Ignite cache:

```csharp
public List<Car> GetCars(string? make, string? model, int? year, SearchMode searchMode, string[]? columns = null)
{
    ICache<int,Car> igniteCache = _igniteService.Cars;
    IQueryable<Car> query = igniteCache.AsCacheQueryable().Select(x => x.Value);
    ...
}
```

Depending on the search mode, we need to build a different query:

```csharp
query = searchMode switch
{
    SearchMode.All => FilterAll(),
    _ => FilterAny()
};
```

When `searchMode` is `All`, it is quite easy to combine multiple `Where` calls to achieve "AND" semantics:

```csharp
IQueryable<Car> FilterAll()
{
    if (make != null)
        query = query.Where(x => x.Make == make);

    if (model != null)
        query = query.Where(x => x.Model == model);

    if (year != null)
        query = query.Where(x => x.Year == year);

    return query;
}
```

However, when `searchMode` is `Any`, we have to build an `Expression` for a single `Where` call, which gets out of hand quickly:

```csharp
IQueryable<Car> FilterAny()
{
    var parameter = Expression.Parameter(typeof(Car), "x");

    Expression? expr = null;

    if (make != null)
        expr = Expression.Equal(Expression.PropertyOrField(parameter, "Make"), Expression.Constant(make));

    if (model != null)
        expr = expr == null
            ? Expression.Equal(Expression.PropertyOrField(parameter, "Model"), Expression.Constant(model))
            : Expression.OrElse(expr, Expression.Equal(Expression.PropertyOrField(parameter, "Model"), Expression.Constant(model)));

    if (year != null)
        expr = expr == null
            ? Expression.Equal(Expression.PropertyOrField(parameter, "Year"), Expression.Constant(year))
            : Expression.OrElse(expr, Expression.Equal(Expression.PropertyOrField(parameter, "Year"), Expression.Constant(year)));

    var expression = Expression.Lambda<Func<Car, bool>>(expr!, parameter);

    return query.Where(expression);
}
```

# Simplification with System.Linq.Dynamic.Core

[Dynamic LINQ (System.Linq.Dynamic.Core)](https://github.com/zzzprojects/System.Linq.Dynamic.Core) 
provides string-based LINQ expression building, which is much easier to use and covers both AND and OR cases:

```csharp
var whereSb = new StringBuilder();
var argIdx = 0;
var args = new List<object>();
AppendArg(make);
AppendArg(model);
AppendArg(year);

query = query.Where(whereSb.ToString(), args.ToArray()); // Ex: make = @0  AND model = @1

void AppendArg(object? value, [CallerArgumentExpression(nameof(value))] string? name = default)
{
    if (value != null)
    {
        if (argIdx > 0)
            whereSb.Append(searchMode == SearchMode.All ? " AND " : " OR ");

        whereSb.Append($"{name} = @{argIdx++} ");
        args.Add(value);
    }
}
```

Full source code, including dynamic column selection, is available on GitHub: [ptupitsyn/ignite-dynamic-linq](https://github.com/ptupitsyn/ignite-dynamic-linq).
You can try it by running [tests](https://github.com/ptupitsyn/ignite-dynamic-linq/blob/main/Ignite.DynamicLINQ.Tests/CarsTests.cs) in your IDE or with `dotnet test`.

Note that [Dynamic LINQ](https://dynamic-linq.net/overview) works with any `IQueryable`, including Ignite.NET's `ICacheQueryable` - 
no specific integration is required. What it does is it parses the string expression and builds an `Expression` tree, 
which is then passed to the LINQ provider to build the SQL query.

We achieved the same result with much less code, which is easier to read and maintain. But is it the right approach?

# Why Not Use SQL Directly?

With Dynamic LINQ library, we are building strings, which are then parsed into `Expression` trees, which are then converted to back to string (SQL).

```
string -> Expression -> string
```

It is an abstraction on top of an abstraction. Some of the strings we pass to Dynamic LINQ library even look like SQL: `make = @0 AND model = @1`.

We can easily adapt the code from above to use SQL directly:

```csharp
var sb = new StringBuilder("SELECT Make, Model, Year FROM Car");

var argIdx = 0;
var args = new List<object>();
AppendArg(make);
AppendArg(model);
AppendArg(year);

return _igniteService.Cars.Query(new SqlFieldsQuery(sb.ToString(), args.ToArray()))
    .Select(x => new Car(
        Val<string>(x, nameof(Car.Make)),
        Val<string>(x, nameof(Car.Model)),
        Val<int>(x, nameof(Car.Year))))
    .ToList();

void AppendArg(object? value, [CallerArgumentExpression(nameof(value))] string? name = default)
{
    if (value != null)
    {
        sb.Append(argIdx++ == 0
            ? " WHERE "
            : searchMode == SearchMode.All ? " AND " : " OR ")
            .Append($"{name} = ? ");

        args.Add(value);
    }
}

T Val<T>(IList<object> row, string name) => cols.IndexOf(name) is var y and >= 0 ? (T)row[y] : default!;
```

The usual caveats apply: no compile-time or IDE checks, and extra care is required to avoid SQL injections.
But we've got rid of 2 layers of abstraction.

# Performance

Let's see how these approaches compare in terms of performance.

```
|      Method | SearchMode |       Mean | Allocated |
|------------ |----------- |-----------:|----------:|
|        Linq |        Any |  45.740 us |  28.46 KB |
| LinqDynamic |        Any |  81.645 us |  61.12 KB |
|         Sql |        Any |   7.035 us |   2.45 KB |
|        Linq |        All | 337.704 us |  53.56 KB |
| LinqDynamic |        All |  80.868 us |  61.13 KB |
|         Sql |        All |   6.969 us |   2.46 KB |
```

Benchmark code is in the same repo: [Program.cs](https://github.com/ptupitsyn/ignite-dynamic-linq/blob/main/Ignite.DynamicLINQ.Benchmarks/Program.cs).

As expected, raw SQL performs much better than LINQ and Dynamic LINQ.

Interestingly, LINQ with `SearchMode.All` is much slower than `SearchMode.Any`, because we use multiple `Where` expressions conditionally.
This is not the case with Dynamic LINQ, which builds a single expression from the provided string.

NOTE: an older post on this blog, [LINQ vs SQL in Ignite.NET: Performance](https://ptupitsyn.github.io/LINQ-vs-SQL-in-Ignite/),
demonstrates that LINQ can be on par with raw SQL, but this requires using compiled queries, which is not possible when dynamic queries are involved.


# More Use Cases

1. Order
2. Custom column set

# Conclusion

This post is inspired by questions coming from Ignite users and GridGain customers. 
Everyone loves LINQ for its ease of use and strong typing, but sometimes it gets in the way.

And when an abstraction gets in the way, it is generally a good idea to drop down one level - in this case, use SQL directly.
