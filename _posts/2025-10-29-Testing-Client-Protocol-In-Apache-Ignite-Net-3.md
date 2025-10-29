---
layout: post
title: How we test client protocol in Apache Ignite.NET
---

Client drivers for distributed systems are much more complicated than traditional database drivers. 
They need to handle data partitioning, cluster topology changes, failover scenarios, and inevitable network issues.

In this post, we'll explore two approaches we use to write integration tests for the Apache Ignite.NET client.

# Proxy Server

Apache Ignite partitions data across multiple nodes in a cluster by computing a hash of the primary key.
Client drivers understand this partitioning logic and send key-based requests directly to the primary node for a given key.

To test this behavior, we implemented a simple proxy server that sits between the client and an actual Ignite cluster.
The proxy decodes client request headers and records requests, allowing us to verify the routing correctness.

Ignite client protocol uses a TCP connection with a frame-based binary protocol. 
Each frame is prefixed with a 4-byte length field, so the proxy can easily read complete frames from the socket and forward them to the cluster.

# Fake Server

TBD

# Links

* TBD
