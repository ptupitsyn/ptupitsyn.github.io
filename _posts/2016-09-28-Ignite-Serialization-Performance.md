---
layout: post
title: Ignite.NET Serialization Performance
---

How fast are different Ignite serialization modes? How do they compare to other popular serializers?

# Reasoning

Ignite is a distributed system, and data serialization is at its core.
Cached data, compute closures, service instances, messages and events - everything is serialized to be transferred over the network.
Therefore, fast and compact serialization is essential for overall system performance.

Ignite uses custom binary protocol, and there is a natural question: is it adequate? How does it compare to standard and third party serializers?

# Size Comparison

# Speed Comparison