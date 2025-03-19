---
layout: post
title: Getting Started with Apache Ignite.NET 3.0
---

Apache Ignite 3.0 has been [released recently](https://ignite.apache.org/blog/whats-new-in-apache-ignite-3-0.html). 
This post is a quick and beginner-friendly guide for .NET developers on how to start with the new version.

# What is Ignite?

In short, [Apache Ignite](https://ignite.apache.org/) is a distributed SQL database. 
As a developer, you work with it as with any other SQL database like PostgreSQL or MySQL, but it is designed to run on multiple nodes and scale horizontally. 
It also provides additional features like distributed computing, data streaming, and more.

Note that Ignite 3.0 is a major release with many changes, so if you are familiar with Ignite 2.x, you will find some new concepts and APIs here.

# Prerequisites

* [.NET SDK](https://dotnet.microsoft.com/en-us/download/dotnet)
* Docker

We will run Ignite in Docker and use the [.NET client](https://ignite.apache.org/docs/ignite3/latest/developers-guide/clients/dotnet) to connect to it.
This guide should work on any OS (Linux, Windows, macOS).

# Start an Ignite Node

Start a node and expose ports `10300` (REST API for management) and `10800` (thin client):

```shell
docker run -p 10300:10300 -p 10800:10800 apacheignite/ignite:3.0.0
```

One node is enough for this guide. We will explore how to scale the cluster in a later post.

# Initialize Ignite Cluster

We have to [initialize the Ignite cluster](https://ignite.apache.org/docs/ignite3/latest/administrators-guide/lifecycle#cluster-initialization) before we can use it.
This can be done with the [CLI tool](https://ignite.apache.org/docs/ignite3/latest/ignite-cli-tool), but to keep this post short, we'll just call the management API directly:

```shell
curl -i --request POST --header "Content-Type: application/json" \
    --data '{"metaStorageNodes": ["defaultNode"], "clusterName": "myCluster"}' \ 
    http://localhost:10300/management/v1/cluster/init
```

**Or** from PowerShell:

```powershell
Invoke-RestMethod -Uri "http://localhost:10300/management/v1/cluster/init" -Method Post -Headers @{"Content-Type"="application/json"} -Body '{"metaStorageNodes": ["defaultNode"], "clusterName": "myCluster"}'
```

**Or** from C# code:

```csharp
await new HttpClient().PostAsync(
    "http://localhost:10300/management/v1/cluster/init",
    JsonContent.Create(new { metaStorageNodes = new[] { "defaultNode" }, clusterName = "myCluster" }));
```

In a moment, our one-node cluster will be initialized and ready to accept client connections on port 10800.

# Connect to Ignite from C#

Create a new project and add the [Apache.Ignite 3.0](https://www.nuget.org/packages/Apache.Ignite/3.0.0) NuGet package:

```shell
dotnet new console
dotnet add package Apache.Ignite --version 3.0.0
```

Replace the contents of `Program.cs` with the following code:

```csharp
using Apache.Ignite;

var cfg = new IgniteClientConfiguration("localhost:10800");
IIgniteClient client = await IgniteClient.StartAsync(cfg);
Console.WriteLine($"Client connected: {client}");
```

Run the project, the output should be:

```
Client connected: IgniteClientInternal { Connections = [ ClusterNode { Id = f6536e19-c9c3-4952-b86d-53b677eb0359, Name = defaultNode, Address = 127.0.0.1:10800 } ] }
```

# Execute SQL Queries

Ignite 3 is SQL-first, so you can interact with it using standard SQL queries.

**Create table** using the standard SQL syntax. The first parameter is transaction; `null` means implicit transaction with autocommit.

```csharp
await client.Sql.ExecuteAsync(null, "CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR)");
```

**Insert data** - note the use of `?` placeholders for parameters:

```csharp
await client.Sql.ExecuteAsync(null, "INSERT INTO test (id, name) VALUES (?, ?)", 1, "John Doe");
```

**Select data**:

```csharp
var resultSet = await client.Sql.ExecuteAsync(null, "SELECT * FROM test");
await foreach (IIgniteTuple row in resultSet)
{
    Console.WriteLine($"Row: {row}");
}
```

The output should be:

```
Row: IgniteTuple { ID = 1, NAME = John Doe }
```

# Conclusion

SQL is one way to interact with Ignite 3.0. 
In the next posts, we will explore other ways to work with data, like key-value APIs, data streamer, object mapping, LINQ, and more.

Stay tuned!

# Links

* [What's New in Apache Ignite 3.0](https://ignite.apache.org/blog/whats-new-in-apache-ignite-3-0.html)
* [Docs](https://ignite.apache.org/docs/ignite3/latest/)
* [Ignite.NET docs](https://ignite.apache.org/docs/ignite3/latest/developers-guide/clients/dotnet)
* [NuGet package](https://www.nuget.org/packages/Apache.Ignite/3.0.0)
