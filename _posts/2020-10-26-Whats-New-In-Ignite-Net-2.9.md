---
layout: post
title: What's new in Apache Ignite.NET 2.8
---

[Apache Ignite](https://ignite.apache.org/) 2.9 [has been released](http://apache-ignite-users.70518.x6.nabble.com/ANNOUNCE-Apache-Ignite-2-9-0-Released-td34311.html) a few days ago.
Let's have a look at .NET-specific features and improvements.  


# Platform Cache: It's All About Performance

* TODO: List included APIs (incl ScanQuery)
* TODO: Compelling benchmarks
* TODO: Usage scenarios (see GG docs - there were some?)
* TODO: Explain like auto-synchronized ConcurrentDictionary


# Call .NET Services From Java: Full Circle of Services

Effectively, .NET services are now first-class citizens and can be called from anywhere: thick and thin clients, Java or .NET.
And, potentially, from other thin clients, like Python or Rust - the protocol supports that (TODO: Links to other clients).

TODO: This works from Java Thin as well!



# IgniteLock: Cache.Lock Replacement

TODO: Cache.Lock should be marked Obsolete 


# Thin Client Automatic Server Discovery

# Thin Client Compute

TODO: Java and .NET services!

# Other Improvements

* SqlFieldsQuery as ContinuousQuery.InitialQuery 
* FieldsQueryCursor metadata
* AffinityCall/AffinityRun with partition - mention this in Platform Cache section instead?
* Thin client cluster APIs - mention in server discovery?


# Wrap-up

TODO:
* Full release notes: https://ignite.apache.org/releases/2.9.0/release_notes.html
* New docs: https://ignite.apache.org/docs/latest/
