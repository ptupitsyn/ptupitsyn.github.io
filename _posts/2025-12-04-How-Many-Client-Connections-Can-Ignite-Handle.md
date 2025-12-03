---
layout: post
title: How many client connections can Ignite 3 handle?
date: 2025-12-04
author: Pavel Tupitsyn
categories: [Apache Ignite]
---

(Spoiler: all of them)

# The Question

A common capacity planning question we get from users is: "How many client connections can one Ignite node maintain?" 

With traditional relational databases, the common knowledge is:
* Client connection is single-threaded and short-lived. "Open -> Do work -> Close" is the typical pattern.
* The server can handle a limited number of concurrent connections.
  * [Postgres defaults to 100](https://www.postgresql.org/docs/current/runtime-config-connection.html#GUC-MAX-CONNECTIONS) `max_connections`.
  * Each connection has significant memory overhead ([a few MBs](https://blog.anarazel.de/2020/10/07/measuring-the-memory-overhead-of-a-postgres-connection/)).
* An external connection pool (like [PgBouncer](https://www.pgbouncer.org/)) is recommended to improve scalability.

[Apache Ignite](https://ignite.apache.org/) is quite different:
* Client connections are long-lived, multiplexed, and thread-safe. Quite often, a single client connection is enough for the entire application lifetime.
* On the server side, each client connection has a small memory footprint (tens of KBs).

This approach with cheap long-lived connections provides low latency and great scalability for applications:
* The connection is always open and responds to queries immediately.
* Multiple queries can be executed concurrently over the same connection (multiplexing).
* No need for an external connection pool.
* Query metadata is cached by the client connection, improving performance for repeated queries.
 

# Testing Setup

* Reduce logging level to WARN to minimize logging overhead.

# Results

![2025-12-04-How-Many-Client-Connections-Can-Ignite-Handle.png](2025-12-04-How-Many-Client-Connections-Can-Ignite-Handle.png)

# Code

https://gist.github.com/ptupitsyn/86056d4143811ba5dde6b2d1704fa948
