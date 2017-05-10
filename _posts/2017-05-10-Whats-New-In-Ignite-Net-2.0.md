---
layout: post
title: What's new in Apache Ignite.NET 2.0
---

Apache Ignite 2.0 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-0-redesigned) last week. Let's see what is new in the .NET part.

![ignite logo](../images/ignite_logo.png)


# Dynamic Type Registration

This is a huge one! In short:

* No need for `BinaryConfiguration` anymore. Types are registered within cluster automatically.
* Everything is serialized in Ignite format, no more limitations for `[Serializable]`, SQL always works.
* **Anything** can be serialized! Classes, structs, system types, delegates, anonymous types, you name it!

# Plugin System

TODO

# LINQ Improvements

TODO

# Other Improvements

TODO

# Breaking Changes

There are some API changes that may break compilation and/or behavior in existing code, please refer to [Migration Guide](https://cwiki.apache.org/confluence/display/IGNITE/Apache+Ignite+2.0+Migration+Guide#ApacheIgnite2.0MigrationGuide-Ignite.NET).