---
layout: post
title: ASP.NET Distributed Output Cache With Apache Ignite
---

Speed up your ASP.NET web farm with a Apache Ignite distributed caching.

![ASP.NET Web Farm with Apache Ignite caching](../images/AspNet/web-farm.png)

# Rationale

ASP.NET performance can be improved in a number of ways, including:

* Cache rendered pages with [Output Cache](https://msdn.microsoft.com/en-us/library/ms178597.aspx).
* Scale out by setting up a [Web Farm](https://technet.microsoft.com/en-us/library/jj129543(v=ws.11).aspx) so that multiple servers handle the requests.

What if we want to combine these approaches? Using default output caching mechanism in a web farm means a separate cache on each server.
So if the load balancer sends a main page request to server 1, and next main page request to server 2, the cache is not used.
If there are N servers, output cache is N times less effective.

Distributed cache solves this issue and provides other benefits:

* All cached data is shared between servers. No matter which server is chosen by the load balancer, it will hit the cache if there is an entry.
* Data is not lost when the worker process recycles.
* RAM from all servers is combined to allow more data to be cached.

Let's see how to set up ASP.NET Output Caching with Apache Ignite.