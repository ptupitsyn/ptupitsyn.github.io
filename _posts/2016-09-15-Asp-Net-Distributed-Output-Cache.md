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

What if we want to combine these approaches?