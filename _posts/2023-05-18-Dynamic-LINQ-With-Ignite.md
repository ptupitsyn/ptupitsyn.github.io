---
layout: post
title: Dynamic LINQ with Ignite.NET and System.Linq.Dynamic.Core
---

TBD: Better title, more catchy? "Dynamic LINQ performance with Ignite.NET and System.Linq.Dynamic.Core"

![Apache Ignite Persistent Store](../images/ignite-dynamic-linq.png) Dynamically building database queries 
can be necessary for some use cases, such as UI-based filtering. 
This can get challenging with LINQ-based frameworks such as EF Core and Ignite.NET.

# Use Case Example

Let's say we have to build a Web API for searching cars with:

```
GET /cars?make=Ford&model=Mustang&searchMode=Any
```

I'm going to use [Apache Ignite.NET](https://ignite.apache.org) as an example, but the same approach can be used with [EF Core](https://learn.microsoft.com/en-us/ef/core/).

The search mode can be `Any` or `All`, and it defines whether we should use `OR` or `AND` in the query. How do we build the query?
Let's start with retrieving IQueryable from Ignite cache:

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

Interestingly, [Dynamic LINQ](https://dynamic-linq.net/overview) works with any `IQueryable`, including Ignite.NET's `ICacheQueryable`. 
What it does is it parses the string expression and builds an `Expression` tree, which is then passed to the LINQ provider to build the SQL query.

We achieved the same result with much less code, which is easier to read and maintain.

# TBD

Does it work? - yes
* Is it a good idea? - not sure
* If an abstraction gets in the way, drop down one level and just build the SQL yourself

* Benchmark? Link to old LINQ vs SQL benchmark, which used compiled queries, but here we can't use compiled!
