---
layout: post
title: .NET Compute in Apache Ignite 3.1
---

Apache Ignite 3.1 has been released last week, and one of the major new features is .NET Compute support. We can now implement compute jobs in .NET, deploy them to the cluster, and call them from any client. Let's try it out!

# Prerequisites

* .NET 8 SDK (to compile .NET jobs)
* Docker (to run Ignite cluster)

# Get the Code

Source code for this walkthrough is available on GitHub [github.com/ptupitsyn/ignite3-dotnet-compute](https://github.com/ptupitsyn/ignite3-dotnet-compute), including the docker-compose file to start the cluster.
