---
layout: post
title: What's new in Apache Ignite.NET 2.11
---

[Apache Ignite](https://ignite.apache.org/) 2.11 [has been released](https://blogs.apache.org/ignite/entry/apache-ignite-2-11-stabilization).
On .NET side there are new examples, thin client DataStreamer, .NET 5 support, and more.

# Data Streamer in Thin Client

Data Streamer API - the most efficient way to load large amounts of data into Ignite - is now available in the .NET thin client. 

Thin streamer automatically buffers the data and groups it into batches for better performance and sends it in parallel to multiple nodes (similarly to the existing "thick" streamer).
If a server node fails, corresponding operations are retried transparently: at-least-once delivery is guaranteed.


TODO: See IEP https://cwiki.apache.org/confluence/display/IGNITE/IEP-68%3A+Thin+Client+Data+Streamer

# Data Streamer API Improvements
TODO

# .NET 5 Support
TODO

# Reworked Examples
TODO

# Conclusion

TODO: Links





TODO:
* Link to release notes, blog post
  * https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt
  * https://blogs.apache.org/ignite/entry/apache-ignite-2-11-stabilization  
* New examples 
  * https://www.nuget.org/packages/Apache.Ignite.Examples/
  * Usage documented https://github.com/apache/ignite/tree/master/modules/platforms/dotnet/examples
* Thin client streamer
* DataStreamer API improv
* .NET 5 support
* 
