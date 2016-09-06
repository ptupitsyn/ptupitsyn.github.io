---
layout: post
title: Building Multi-Platform Ignite Cluster&#58; Java + .NET
---

Ignite cluster can consist of nodes on any supported platform: Java, .NET and C++. Let's see how to run .NET/Java cluster with NuGet and Maven.

![Apache Ignite Java + .NET](../images/multi-platform-cluster.png)

# Prerequisites

This post is both for .NET developers who may be new to Java and vice versa, so I'll provide very detailed steps.

We are going to use the following software:

* Visual Studio 2015 (includes NuGet; free Community edition)
* IntelliJ IDEA (includes Maven; free Community edition)

Complete source code for this post is available on GitHub: [github.com/ptupitsyn/ignite-multi-platform-demo](https://github.com/ptupitsyn/ignite-multi-platform-demo)

# Goals

* Connect Java and .NET node
* Share data through Ignite cache using Java and .NET classes with the same name and fields
* Run continuous query to see real-time updates from another platform

# Setting Up Java Project

* Run IntelliJ IDEA and click "Create New Project":

![IDEA Welcome Screen](../images/Multi-Platform-Cluster/idea1.png)

* Select Maven and click "Next":

![IDEA New Project Screen](../images/Multi-Platform-Cluster/idea2.png)

* Enter Maven info, click "Next" and "Finish":

![IDEA Maven Project Screen](../images/Multi-Platform-Cluster/idea3.png)
![IDEA Maven Project Screen](../images/Multi-Platform-Cluster/idea4.png)

* As a result, you should see the `pom.xml` file of the new project open:

![IDEA Main Screen](../images/Multi-Platform-Cluster/idea5.png)

* Add Ignite dependency to the `<project>` section:

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.ignite</groupId>
        <artifactId>ignite-core</artifactId>
        <version>1.7.0</version>
    </dependency>
</dependencies>
```

* IDEA may ask you to import project changes, click any of the links:

![IDEA Import Changes](../images/Multi-Platform-Cluster/maven-auto-import.png)

* Add `Demo` class to src\main\java with the following code:

```java
import org.apache.ignite.Ignition;

public class Demo {
    public static void main(String[] args) {
        Ignition.start();
    }
}
```

* Run it with `Shift+F10` and verify that node starts in IDEA console:

![IDEA Test Program](../images/Multi-Platform-Cluster/idea6.png)

* Stop the program with `Ctrl+F2` or the stop button.

# Setting Up .NET Project

* Run Visual Studio and click File -> New -> Project
* Select Visual C# -> Windows -> Console Application
* Make sure to have .NET Framework of version 4+ selected on top

 ![VS New Project](../images/Multi-Platform-Cluster/vs1.png)

* Click "OK", and you will be presented with an empty console project
* Bring up NuGet console: Menu -> Tools -> NuGet Package Manager -> Package Manager Console
* Type `Install-Package Apache.Ignite`

![VS Install Package](../images/Multi-Platform-Cluster/vs2.png)

* Hit Enter and there should be a `Successfully installed 'Apache.Ignite 1.7.0' to IgniteMultiPlatform` message.
* Replace `Program.cs` contents with the following code:

```cs
using System;
using Apache.Ignite.Core;

class Program
{
    static void Main(string[] args)
    {
        Ignition.Start();
        Console.ReadKey();
    }
}
```

* Run the program with `Ctrl-F5` and verify that it starts Ignite in console:

![Console Test](../images/Multi-Platform-Cluster/vs3.png)

# Tweak Java Node Configuration To Understand .NET Node

Now if you run both Java node in IDEA and .NET node in Visual Studio, you will get the following error in one of them:

```text
IgniteSpiException: Local node's binary configuration is not equal to remote node's binary configuration [locBinaryCfg={globSerializer=null, compactFooter=true, globIdMapper=org.apache.ignite.binary.BinaryBasicIdMapper}, rmtBinaryCfg=null]
```

The problem here is that .NET nodes only support `BinaryBasicIdMapper` and `BinaryBasicNameMapper` in `BinaryConfiguration`, and we should set them explicitly in Java.
Replace `Ignition.start();` line with the following code:

```java
BinaryConfiguration binCfg = new BinaryConfiguration();

binCfg.setIdMapper(new BinaryBasicIdMapper());
binCfg.setNameMapper(new BinaryBasicNameMapper());

IgniteConfiguration cfg = new IgniteConfiguration().setBinaryConfiguration(binCfg);

Ignition.start(cfg);
```

Run both .NET and Java nodes and verify that they join each other:

```text
[15:04:17] Topology snapshot [ver=2, servers=2, clients=0, CPUs=8, heap=7.1GB]
```

# Exchange Data Through Ignite cache

TODO: Chat application with similar code in both platforms
* use Message class with Author and Text fields