---
layout: post
title: Entity Framework As Ignite.NET Cache Store
---

Implement persistent store with Entity Framework and SQL Server.


# What Is Persistent Store

Apache Ignite stores cached data in RAM.
Fault tolerance can be achieved by [configuring caches in a certain way](https://apacheignite.readme.io/docs/cache-modes).

However, there are use cases where disk-based storage may be needed:

* Store all data on disk so it survives cluster restarts.
* Use Ignite as a cache in front of disk-based RDBMS to improve performance.

Cache store is represented by `ICacheStore` interface,
which is called by Ignite when certain `ICache` methods are called [(more details)](https://apacheignite-net.readme.io/docs/persistent-store).

In this post we are going to create `ICacheStore` implementation that delegates all operations to the EntityFramework.


# Code

The full source code is on [github.com/ptupitsyn/ignite-net-examples](https://github.com/ptupitsyn/ignite-net-examples), under EFCacheStore folder.

The project is self-sufficient, you can download the sources and run it without setting anything up.
It uses SQL Server Compact (via NuGet) and creates a database in the bin folder when needed.


# Data Model

* We are going to use Entity Framework Code First approach to define the data model.
* The same classes will be used in Ignite caches.

Other approaches are possible. Ignite cache store can work in binary mode and use raw SQL to store and retrieve data from an RDBMS, for example.

There are a couple of gotchas with using EF model classes in Ignite cache without modification:

* By default, EF returns from `IDbSet` proxy objects of anonymous generated classes, which can not be serialized by Ignite. To disable this, set `DbContext.Configuration.ProxyCreationEnabled` to `false`.
* By default, `CacheConfiguration.KeepBinaryInStore` is true in Ignite, which means that ICacheStore methods receive IBinaryObject instances instead of actual objects. We want to use model classes directly, so this has to be disabled as well.
* Model classes have to be registered in BinaryConfiguration to enable Ignite serialization for them.
* Database-generated ids may have to be disabled when you reuse id values as Ignite cache keys.

The data model itself is a simple one-to-many relationship of Blogs and Posts:

```cs
public class Blog
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.None)]
    public int BlogId { get; set; }
    public string Name { get; set; }
    public virtual List<Post> Posts { get; set; }
}

public class Post
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.None)]
    public int PostId { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public int BlogId { get; set; }
    public virtual Blog Blog { get; set; }
}
```


# Implementing Cache Store

The "meat" of the `ICacheStore` interface are `Load`, `Write` and `Delete` methods, which are called when related operations are invoked on `ICache`.

For example, when `ICache.Get(1)` is called, and there is no cache entry with specified key in Ignite cache,
Ignite calls `ICacheStore.Load` method, which can be implemented like this:

```cs
public object Load(object key)
{
    using (var ctx = new BloggingContext
    {
        Configuration = { ProxyCreationEnabled = false }
    })
    {
        Blog blog = ctx.Blogs.Find(key);

        return blog;
    }
}
```

Here we ask Entity Framework context for the Blog with the key `1` and return it back to Ignite.
Ignite then stores the value in cache and returns it to the caller.
Subsequent requests to the key `1` will be served from Ignite cache directly, without cache store invocation.

Since we reuse the same model class, we can return it directly from EF to Ignite.
Navigation properties (`Blog.Posts` and `Post.Blog`) are not populated (`null`) when `ProxyCreationEnabled` is set to `false`, so Ignite won't cache irrelevant data.


# Setting Up Cache Store

Persistent store can be enabled for each Ignite cache separately via `CacheConfiguration.CacheStoreFactory` property.
CacheStoreFactory requires `IFactory<ICacheStore>` instance, which is implemented like this:

```cs
[Serializable]
public class BlogCacheStoreFactory : IFactory<ICacheStore>
{
    public ICacheStore CreateInstance()
    {
        return new BlogCacheStore();
    }
}
```

ICacheStore implementation is not serialized by Ignite, so there are no requirements for BinaryConfiguration or `[Serializable]`.
Instead, CacheStoreFactory is serialized when cache is deployed to remote nodes, and it should be `[Serializable]` (no option for Ignite serialization here).

Other `CacheConfiguration` properties to be enabled for cache store to work are 
`ReadThrough`, `WriteThrough` and `WriteBehind` (see [(docs)](https://apacheignite-net.readme.io/docs/persistent-store) for more details).

So the resulting configuration for our case look like this:

```cs
new CacheConfiguration
{
    CacheStoreFactory = new BlogCacheStoreFactory(),
    ReadThrough = true,
    WriteThrough = true,
    KeepBinaryInStore = false
}
```