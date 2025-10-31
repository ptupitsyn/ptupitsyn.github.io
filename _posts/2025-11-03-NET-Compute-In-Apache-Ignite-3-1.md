---
layout: post
title: .NET Compute in Apache Ignite 3.1
---

Apache Ignite 3.1 has been released last week, and one of the major new features is .NET Compute support. We can now implement compute jobs in .NET, deploy them to the cluster, and call them from any client. Let's try it out!

# Run the Demo

Code first - talk later.

1. Clone:

```shell
git clone https://github.com/ptupitsyn/ignite3-dotnet-compute
cd ignite3-dotnet-compute
```

2. Start the Ignite cluster:

```shell
docker-compose up -d
```

3. Run the demo:

```shell
cd IgniteCompute
dotnet run
```

# How It Works

In the demo, we have:
* Compiled a .NET library with compute jobs (`IgniteCompute.Jobs.dll`)
* Deployed it to the cluster via the REST API (`ManagementApi.cs`)
* And invoked the jobs from a .NET client.
