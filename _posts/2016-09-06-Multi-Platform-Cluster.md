---
layout: post
title: Building Multi-Platform Ignite Cluster&#58; Java + .NET
---

Ignite cluster can consist of nodes on any supported platform: Java, .NET and C++. Let's see how to run .NET/Java cluster with NuGet and Maven.

![Apache Ignite Java + .NET](../images/multi-platform-cluster.png)

# Prerequisites

This post is both for Java developers who need to run Ignite.NET and .NET developers who need to run Ignite on Java,
so I'll try to provide detailed steps on both sides.

We are going to use the following software:

* Visual Studio 2015 (includes NuGet; free Community edition)
* IntelliJ IDEA (includes Maven; free Community edition)
* Java 8

# Goals

* Connect Java and .NET node
* Share data through Ignite cache using Java and .NET classes with the same name and fields
* Run continuous query to see real-time updates from another platform
* Call Java service from .NET

# Setting Up Java Project