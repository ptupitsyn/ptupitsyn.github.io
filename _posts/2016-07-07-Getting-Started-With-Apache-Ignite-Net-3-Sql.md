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


---
// TODO:
How SQL queries work; H2 Debug Console
