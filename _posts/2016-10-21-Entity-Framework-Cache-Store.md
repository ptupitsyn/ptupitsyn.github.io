---
layout: post
title: Entity Framework As Ignite.NET Cache Store
---

Implement Ignite.NET persistent store with Entity Framework and SQL Server.


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

There are a couple of gotchas with using EF model classes in Ignite cache:

* By default, EF queries return proxy objects of runtime-generated classes, which can not be serialized by Ignite. To disable this, set `DbContext.Configuration.ProxyCreationEnabled` to `false`.
* By default, `CacheConfiguration.KeepBinaryInStore` is true in Ignite, which means that `ICacheStore` methods receive `IBinaryObject` instances instead of actual objects. We want to use model classes directly, so this has to be disabled as well.
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

`ICacheStore` implementation is not serialized by Ignite, so there are no requirements for `BinaryConfiguration` or `[Serializable]`.
Instead, `CacheStoreFactory` is serialized when cache is deployed to remote nodes, and it should be `[Serializable]` (no option for Ignite serialization here).

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


# Running The Example

I've added a bunch of `WriteLine` calls to better understand what happens when we work with Ignite cache with cache store enabled.

First, we initialize the SQL database with one Blog and one Post

```text
Database created at C:\W\ignite-net-examples\EFCacheStore\IgniteEFCacheStore\bin\Debug\blogs.db with 2 entities.
```

Then, we call `LoadCache` on `posts` cache, but not on `blogs` cache.
This call delegates to `ICacheStore.LoadCache()` which loads all Posts from SQL db into Ignite cache.

```text
Calling ICache.LoadCache...
PostCacheStore.LoadCache() called.
```

After that we iterate over Post entities in Ignite cache and retrieve related Blog entities.
Since Blogs were not preloaded, each `ICache.Get` delegates to `ICacheStore.Load()` method:

```text
>>> List of all posts:
Retrieving blog with id 0...
BlogCacheStore.Load(0) called.
>>> Post 'Getting Started With Ignite.NET' in blog 'Ignite Blog'
>>> End list.
```

Then we add a new Post entity to Ignite cache, which propagates to `ICacheStore.Write` to persist the new entity in SQL db:

```text
Adding new post to existing blog..
PostCacheStore.Write(1, IgniteEFCacheStore.Post) called.
```

When iterating over Posts again, Ignite no longer delegates `blogs.Get` calls to cache store, since Blog entity is already in Ignite memory:

```text
>>> List of all posts:
Retrieving blog with id 0...
>>> Post 'Getting Started With Ignite.NET' in blog 'Ignite Blog'
Retrieving blog with id 0...
>>> Post 'New Post From Ignite' in blog 'Ignite Blog'
>>> End list.
```

Lastly, we remove the newly created Post, which also deletes it from SQL database:

```text
Removing post with id 1...
PostCacheStore.Delete(1) called.
```


# Generic EF Cache Store

Looking at `BlogCacheStore` and `PostCacheStore` classes, one can notice that they are very similar.
My attempt at generifying the implementation can be found in [`EntityFrameworkCacheStore<TEntity, TContext>`](https://github.com/ptupitsyn/ignite-net-examples/blob/master/EFCacheStore/IgniteEFCacheStore/EntityFrameworkCacheStore.cs) class.
It is far from perfect (for example, LoadAll is inefficient), but you get the idea.


# Update: EF Performance

As [mirhagk noted on Reddit](https://www.reddit.com/r/programming/comments/593ep1/entity_framework_as_ignitenet_cache_store/d95mnt6/),
Entity Framework incurs quite an overhead. This post is just an example of Ignite cache store; EF is used because it is well-known and easy.

*I would thoroughly recommend you do not use entity framework in instances where performance is critical.
Let me be clear, EF is fantastic at doing what it's meant to do, but most people will never use all the features it has and it's quite heavy because of those features.
Something like PetaPoco is orders of magnitude faster though, not just because of tiny amount of overhead but because it forces you to think through what you are doing. I'm not just talking about joins (which aren't nearly as expensive as people make them out to be, as long as you use a real database), but the lazy loading features of EF makes it very easy to write code that hits the databases hundreds of times in a web request, and no matter how much EF might optimize the latency to the database is going to dominate all other performance concerns.*