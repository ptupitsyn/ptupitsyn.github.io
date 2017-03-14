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

Besides being easier to use, `TransactionScope` API allows performing transactional operations across multiple systems (via [Two Phase Commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)).



Documentation page: https://apacheignite-net.readme.io/docs/transactionscope-api

# Distributed DML

Documentation page: https://apacheignite-net.readme.io/docs/distributed-dml

# LINQ Improvements

Distributed LINQ continues to evolve:

## Contains

## DateTime Properties

## Inline Joins
