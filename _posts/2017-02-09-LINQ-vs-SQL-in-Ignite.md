---
layout: post
title: LINQ vs SQL in Ignite.NET
---

LINQ has many benefits over SQL, but at what cost?

# How Ignite LINQ Works

Let's say we have the following equivalent queries:

```cs
var orgs = ignite.GetCache<int, Organization>("orgs");




```



The code can be found at https://github.com/ptupitsyn/IgniteNetBenchmarks