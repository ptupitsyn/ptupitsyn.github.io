---
layout: post
title: Getting Started With Apache Ignite.NET 3.0 alpha 3
---

Apache Ignite 3.0 alpha 3 [has been released last week](https://lists.apache.org/thread.html/r762caa7fe3f52f2faaae660024bb52de17b6f06eacba695b229d847d%40%3Cdev.ignite.apache.org%3E). 
This release introduces .NET and Java thin clients for the first time in 3.0. 
Let's see how to get started with .NET thin client. 


# What is Ignite 3

3.0 is the next major Ignite version. It aims to provide improved developer experience with better architecture, APIs, and tooling.

Check documentation and blogs for more details:
* https://ignite.apache.org/docs/3.0.0-alpha/index
* https://www.gridgain.com/resources/blog/ignite-3-alpha-sneak-peek-future-apache-ignite
* https://www.gridgain.com/resources/blog/apache-ignite-3-alpha-3-apache-calcite-raft-and-lsm-tree

Note that 3.0 is still in alpha state, it is developed in parallel with Ignite 2.x, which will continue to evolve for years to come.  


# Prerequisites

* [JDK 11](https://openjdk.java.net/install/)
* [.NET 5 or .NET Core 3.1 SDK](https://dotnet.microsoft.com/download/dotnet)
* [Maven 3](https://maven.apache.org/download.cgi)

This guide should work on any modern OS (Linux, Windows, macOS).


# Install Ignite CLI

Ignite CLI tool is the new way to start, stop, and manage server nodes.

* Download release binaries: [apache-ignite-3.0.0-alpha3.zip](https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=ignite/3.0.0-alpha3/apache-ignite-3.0.0-alpha3.zip).
* Unzip to a folder and open your favorite terminal there.
* Run `./ignite init` to install the artifacts.

NOTE: in case something