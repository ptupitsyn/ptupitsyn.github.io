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
* Deployed it to the cluster via the REST API (`ManagementApi.DeployAssembly(...)`)
* And invoked the jobs from a .NET client (`client.Compute.SubmitAsync(...)`).

This is possible because Ignite docker image includes the .NET runtime and a dedicated Compute Executor for .NET jobs.
When we say `var helloJobDesc = JobDescriptor.Of(new HelloJob())`, Ignite sets the executor type for us, which looks like this in long form:

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

Another big improvement is explicit deployment and versioning:
* We have full control over deployment and undeployment with the Management API (available as a REST API or a CLI tool).
* We can deploy multiple versions of the same assembly and choose which version to use when submitting a compute job.
* Deployment units are isolated from each other, so there are no assembly conflicts.

# Implementing a Compute Job

Key points:
* Place the job types in a separate assembly (DLL) 
  * The assembly must be a "class library" (created with `dotnet new classlib`).
  * [Ignite 3.1.0 docker image](https://hub.docker.com/layers/apacheignite/ignite/3.1.0/images/sha256-d26bbc56d26f5483b596e966e05af7ddd16cf0cd70b8dd38c4ff418d69e4f520) includes .NET 8 runtime, so target `net8.0`.
    * With custom images or deployments, newer .NET versions can be used as well.
* Reference the `Apache.Ignite` NuGet package (v3.1.0 or later).
  * The referenced version does **not** have to match the server version. As long as the job uses supported APIs, it can run on older or newer server versions.
* Implement the `IComputeJob<TArg, TResult>` interface.
* Job types must be public and have a public parameterless constructor.
* Implement `IDisposable` and/or `IAsyncDisposable` if you need to clean up resources, Ignite will call `Dispose`/`DisposeAsync` after job execution.

# Job Execution Modes

## Single Node

Execute a job on a specific node or any node from a given set using `JobTarget.Node(IClusterNode node)` and `JobTarget.AnyNode(params IClusterNode[] nodes)` 

## Colocated

Execute a job on the node where a specific primary key is located using `JobTarget.Colocated<TKey>(QualifiedName tableName, TKey key)`

## Broadcast

Execute a job on all specified nodes using `BroadcastJobTarget.Nodes(params IClusterNode[] nodes)` and get results from all nodes.

# User Scenarios

## Send Code to Data

In a traditional scenario, we load some data from a database, process it in the client application, and write the results back to the database.
This approach has drawbacks:
* Data transfer overhead - moving large datasets over the network is slow.
* Increased lock contention due to longer transactions.
* Limited scalability - the client application can become a bottleneck.

With colocated compute jobs (`JobTarget.Colocated`), we invert the scenario by invoking processing logic directly on the server node where the data resides:
* Job descriptors and parameters are small, so network overhead is minimal.
* Only one round-trip is needed to perform any number of data reads and writes.
* Processing happens close to the data, reducing lock contention.

## MapReduce

Split a large task into smaller subtasks, execute them in parallel on multiple nodes (`BroadcastJobTarget`), and aggregate the results.

## Language Interoperability

TBD

## Inter-node Communication

TBD

# Conclusion

TBD
