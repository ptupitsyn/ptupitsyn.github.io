---
layout: post
title: What's new in Apache Ignite.NET 2.14
---

[Apache Ignite](https://ignite.apache.org/) 2.14 [has been released](https://lists.apache.org/thread/s4l32s59no8lx689y921o47xdtg8g3n3).
.NET updates include Service Interceptors and AtomicLong.

# Service Interceptors (Middleware)

[Ignite Services API](https://ignite.apache.org/features/service-apis.html) provides a powerful mechanism for implementing distributed service-oriented applications.
Those applications typically need to deal with a number of [cross-cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern), such as logging, monitoring, authentication/authorization, validation, and so on.

Adding the same logging and monitoring code to every service method is tedious and error-prone. Service Interceptors are designed to solve this.
We took inspiration from [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-6.0) and [gRPC Interceptors](https://grpc.io/blog/grpc-web-interceptor/)
to create [IServiceCallInterceptor](https://ignite.apache.org/releases/latest/dotnetdoc/api/Apache.Ignite.Core.Services.IServiceCallInterceptor.html) API, 
which provides full control over service method execution.

You can add logic before and after the actual method, handle exceptions, or skip the method call completely:

```csharp
class MyInterceptor : IServiceCallInterceptor
{
    public object Invoke(string mtd, object[] args, IServiceContext ctx, Func<object> next)
    {
        Console.WriteLine($"Before {mtd}");

        try
        {
            // Invoke the next interceptor in the chain, or the actual method.
            var res = next.Invoke();
            Console.WriteLine($"{mtd} returned {res}");
            return res;
        }
        catch (Exception e)
        {
            Console.WriteLine($"Exception {e} in {mtd}");
            throw;
        }
        finally
        {
            Console.WriteLine($"After {mtd}");
        }
    }
}
```

 

TODO: Cross-cutting concerns

More details:

* TODO: IEP, APIs

# AtomicLong in Thin Client 

More details:

* TODO: IEP, APIs


# Links

* Full release notes: [https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt](https://github.com/apache/ignite/blob/master/RELEASE_NOTES.txt)
* Download: [https://ignite.apache.org/download](https://ignite.apache.org/download.cgi)
