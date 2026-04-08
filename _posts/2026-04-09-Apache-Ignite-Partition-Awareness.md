---
layout: post
title: Partition Awareness in Apache Ignite - Detailed Guide
date: 2026-01-29
author: Pavel Tupitsyn
categories: [Apache, Ignite, Client, Partitioning, Sharding, Performance]
---

Everything you need to know about partition awareness in Apache Ignite.

# What is Partition Awareness

Partition awareness is a feature of Apache Ignite clients that allows them to send requests directly to the cluster node that owns the relevant data partition, instead of going through a random node. 
This can significantly reduce latency and improve performance for data access operations.

Simply, the client knows how to find the right node for a given key.

# Key Ideas

- Ignite splits data into partitions and distributes them across cluster nodes.
- `partition(key) = hash(key) % partitionCount` - fast, simple and local formula.
- `node(partition) = partitionAssignment[partition]` - client retrieves and keeps updated a mapping of which node owns which partition.
- Connection management, partition awareness, and failover are three separate concerns. The client handles them independently.

## Connection Management

* The client has a list of server addresses.
* As long as at least one connection is alive, the client is operational.
* Any client request can be sent to any server node. The server will forward it to the right node if needed.
