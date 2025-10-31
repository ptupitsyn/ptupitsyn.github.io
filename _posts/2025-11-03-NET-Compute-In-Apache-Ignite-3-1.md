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
* Started a 3-node Ignite cluster in Docker (`docker-compose.yml`)
* Compiled a .NET library with compute jobs (`IgniteCompute.Jobs.dll`)
* Deployed it to the cluster via the REST API (`ManagementApi.cs`)
* And invoked the jobs from a .NET client.

This is possible because Ignite docker image includes the .NET runtime and a dedicated Compute Executor for .NET jobs.
When we say `var helloJobDesc = JobDescriptor.Of(new HelloJob())`, Ignite sets the executor type for us, which looks like this under the hood:

```csharp
var helloJobDesc = new JobDescriptor<string, string>(typeof(HelloJob).AssemblyQualifiedName!)
{
    Options = new JobExecutionOptions
    {
        ExecutorType = JobExecutorType.DotNetSidecar
    }
};
```

`DotNetSidecar` executor starts a separate .NET process on the server node, which loads the job assembly and executes the job.
* The process is started on-demand when the first .NET job is executed on the node.
* The process is reused for subsequent job executions.

This is different from the Ignite 2.x approach, where CLR and JVM were hosted in the same process. The benefits of the new approach are:
* Better isolation between .NET and Java runtimes.
  * If the process crashes, it doesn't bring down the whole Ignite node, and it will be restarted automatically.
* No overhead if there are no .NET jobs.
* Other executors (Python, C++, Wasm) can coexist on one node without interfering with each other.
