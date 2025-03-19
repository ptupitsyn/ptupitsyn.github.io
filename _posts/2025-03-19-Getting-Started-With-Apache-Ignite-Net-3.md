---
layout: post
title: Getting Started with Apache Ignite.NET 3.0
---

Apache Ignite 3.0 has been released recently, and it differs quite a lot from previous versions.

This post is a quick and beginner-friendly guide for .NET developers on how to start with Ignite 3.0.

# What is Apache Ignite?

In short, [Apache Ignite](https://ignite.apache.org/) is a distributed SQL database. 
As a developer, you work with it as with any other SQL database like PostgreSQL or MySQL, but it is designed to run on multiple nodes and scale horizontally. 

# Prerequisites

* [.NET SDK](https://dotnet.microsoft.com/en-us/download/dotnet)
* Docker

We will run Ignite in Docker and use the [.NET client](https://ignite.apache.org/docs/ignite3/latest/developers-guide/clients/dotnet) to connect to it.

# Start an Ignite Node

Start a node and expose ports 10300 (REST API) and 10800 (thin client):

```shell
docker run -p 10300:10300 -p 10800:10800 apacheignite/ignite:3.0.0
```

One node is enough for this guide. We will explore how to scale the cluster in a later post.

# Initialize Ignite Cluster

We have to [initialize the Ignite cluster](https://ignite.apache.org/docs/ignite3/latest/administrators-guide/lifecycle#cluster-initialization) before we can use it.
This can be done with the [CLI tool](https://ignite.apache.org/docs/ignite3/latest/ignite-cli-tool), but the quickest way is to issue a POST request to the `management/v1/cluster/init` endpoint:

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
