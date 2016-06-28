---
layout: post
title: Getting Started with Apache Ignite.NET Part 2: Distributed Cache
---

In this part we will add data to the distributed cache, perform atomic operations, and retrieve data with SQL and LINQ.

* [Part 1: Getting Started](https://ptupitsyn.github.io/Getting-Started-With-Apache-Ignite-Net/)
* Part 2: Distributed Cache  

## What is Ignite Cache
You can think of Ignite cache as `Dictionary<K, V>` where entries are distributed across multiple machines. 
In `Partitioned` cache mode, each Ignite node stores only a part of all cache entries. However, all cache entries can be accessed by any given node at any moment. 