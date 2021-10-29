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

**NOTE:** in case something goes wrong and you'd like to start from scratch:
* Delete `.ignitecfg` from your home directory.
* Delete binaries folder and unzip the binaries to a new one.


# Start a Server Node

All commands here and below should be run in the binaries directory.

* Run `./ignite node start --config=examples/config/ignite-config.json my-first-node`
* Confirm the node is started and running: `./ignite node list`

The server is now running in the background.


# Populate Sample Data

We are going to use built-in Java examples to initialize a test Ignite table, because `CreateTable` API is not yet available in thin clients.

* Comment out the last line with `dropTable` in `examples/src/main/java/org/apache/ignite/example/table/KeyValueViewExample.java`
* `cd examples`
* `mvn compile exec:java -Dexec.mainClass="org.apache.ignite.example.table.KeyValueViewExample"`

As a result, we'll have a table named `PUBLIC.accounts` on the server.


# Create .NET Project

* `dotnet new console -o IgniteDotNetExample`
* `cd IgniteDotNetExample`
* `dotnet add package Apache.Ignite --version 3.0.0-alpha3`


# Connect .NET Client

Replace `Program.cs` contents with the following:

```cs
using System;
using Apache.Ignite;
using Apache.Ignite.Log;
using Apache.Ignite.Table;

var cfg = new IgniteClientConfiguration("127.0.0.1:10800")
{
    Logger = new ConsoleLogger { MinLevel = LogLevel.Trace }
};

using IIgnite client = await IgniteClient.StartAsync(cfg);
```

Run the program with `dotnet run` or in your IDE, it should print something like

```
Socket connection established: [::ffff:127.0.0.1]:60622 -> [::ffff:127.0.0.1]:10800
```


# Try Table API

Print all table names:

```cs
IList<ITable> tables = await client.Tables.GetTablesAsync();

foreach (var t in tables)
    Console.WriteLine(t.Name);
```

Get table by name:

```cs
ITable table = await client.Tables.GetTableAsync("PUBLIC.accounts");
Console.WriteLine("Table exists: " + (table != null));
```

Insert row:

```cs
var row = new IgniteTuple
{
    ["accountNumber"] = 101,
    ["balance"] = (double)300,
    ["firstName"] = "First",
    ["lastName"] = "Last"
};

await table.UpsertAsync(row);
```

Get row by key:

```cs
var key = new IgniteTuple
{
    ["accountNumber"] = 101
};

IIgniteTuple val = await table.GetAsync(key);
Console.WriteLine(val);
```

...which should print:
```
IgniteTuple [accountNumber=101, balance=300, firstName=First, lastName=Last]
```

If you are familiar with Ignite 2.x, we can draw some parallels:
* `ICache` -> `ITable`
* `IBinaryObject` -> `IgniteTuple`


# Stop the Cluster

* `./ignite node stop my-first-node`
