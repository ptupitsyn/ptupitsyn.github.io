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

# Goals

* Connect Java and .NET node
* Share data through Ignite cache using Java and .NET classes with the same name and fields
* Run continuous query to see real-time updates from another platform
* Call Java service from .NET

# Setting Up Java Project

Run IntelliJ IDEA and click "Create New Project":

![IDEA Welcome Screen](../images/Multi-Platform-Cluster/idea1.png)

Select Maven and click "Next":

![IDEA New Project Screen](../images/Multi-Platform-Cluster/idea2.png)

Enter Maven info, click "Next" and "Finish":

![IDEA Maven Project Screen](../images/Multi-Platform-Cluster/idea3.png)
![IDEA Maven Project Screen](../images/Multi-Platform-Cluster/idea4.png)

As a result, you should see the `pom.xml` file of the new project open:

![IDEA Main Screen](../images/Multi-Platform-Cluster/idea5.png)

Add Ignite dependency to the `<project>` section:

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.ignite</groupId>
        <artifactId>ignite-core</artifactId>
        <version>1.7.0</version>
    </dependency>
</dependencies>
```

IDEA may ask you to import project changes, click any of the links:

![IDEA Import Changes](../images/Multi-Platform-Cluster/maven-auto-import.png)

Add `Demo` class to src\main\java with the following code: 

```java
import org.apache.ignite.Ignition;

public class Demo {
    public static void main(String[] args) {
        Ignition.start();
    }
}
```

Run it with `Shift+F10` and verify that node starts in IDEA console:

![IDEA Test Program](../images/Multi-Platform-Cluster/idea6.png)