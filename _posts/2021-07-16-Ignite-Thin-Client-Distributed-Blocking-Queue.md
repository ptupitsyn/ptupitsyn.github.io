---
layout: post
title: Apache Ignite Thin Client Distributed Blocking Queue
---

Apache Ignite "thick" API provides a number [Distributed Data Structures](https://ignite.apache.org/features/datastructures.html), such as Queues and Atomics. Those APIs are not yet available in thin clients, but we can easily implement them on top of the Cache API.     


# Distributed Blocking Queue

TODO: What and why

# Closing

* Add cache configuration
* Implement interfaces: `IEnumerable`, `ICollection`, `IProducerConsumerCollection`
