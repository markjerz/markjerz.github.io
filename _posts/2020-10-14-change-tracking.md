---
title: A survey of change tracking techniques in .Net
---

# Introduction

I have been recently thinking about how one would make a pure repository implementation (see [Repository Pattern C#](https://codewithshadman.com/repository-pattern-csharp/) for details), of the type recommended in the DDD books. The desired approach to repository creation in DDD is to facilitate the de-coupling of the infrastructure code (e.g. accessing a database) from the domain specific code (e.g. creating a new entity). In order to achieve that, they say, you should create repositories that look very much like in-memory collections.

In practice that means an interface that roughly looks like this:

```
public interface IRepository<T> {
    Task<T> GetByIdAsync<TKey>(TKey id);

    Task<IEnumerable<T>> GetAllAsync();

    void Add(T entity);    

    void Delete(T entity);
}
```

If we look at the code from [Repository Pattern C#](https://codewithshadman.com/repository-pattern-csharp/) we can immediately see some clear differences...

```
public interface ICustomerRepository {        
    IEnumerable GetCustomers();        

    Customer GetCustomerByID(int customerId);        

    void InsertCustomer(Customer customer);        

    void DeleteCustomer(int customerId);        

    void UpdateCustomer(Customer customer);        

    void Save();    
}
```

Obviously, we have the difference in that our interface uses Tasks (a necessary evil of async code) but we have two other fundamental differences:

1. There is a `Save()` method! Now, in DDD, this has no business being inside the domain. This is clearly an infrastructure level concern, in that it requires you to save changes to the persistence. However, our domain code should not concern itself with when to save changes to the database.

2. There is an `UpdateCustomer(Customer customer)` method. This doesn't adhere to our other requirement, that we should model the repository as an in-memory collection. When you make changes to an instance inside a `List<T>` (for example) you don't have to tell the list they've been updated - as long as T is a reference type the list will just contain references to the entity that you've updated, no need for an explicit notification.

So, given that we are no longer able to tell the repository when to update or save changes how do we do this automatically as part of the infrastructure?

**We need a change tracker!** - a thing that will tell us when changes have been made, and what, do that we can automatically update the persistence layer.

Many .Net [ORM](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)s provide similar functionality. This post is a review of some of the techniques used to implement it.

## EF Core/NHibernate

While EF Core does support multiple change trackers, it advocates the same functionality that NHibernate uses: they copy values during load of the entity, hold references to the entity, and then, when you want to persist all changes, it compares all loaded entities with their intial values to identify the changes before then persisting those changes.

Looking at a simple example from EF Core:

```
using (var context = new BloggingContext())
{
    var blog = context.Blogs.First();
    blog.Url = "http://example.com/blog";
    context.SaveChanges();
}
```

The key parts of this are that when you call `context.Blogs.First()` EF executes a sql query (assuming not already in the session) to get the blog data from SQL, it then does two things: store the properties in an actual instance of `Blog`, which it returns from this method invocation, and it stores the properties in a map so that it knows the original values.

When you change the `Url` property nothing happens as far as EF is concerned. This is just a standard property setter.

When you then execute `context.SaveChanges()` EF looks at all the entities loaded through the context, finds a single Blog entry, does a property by property comparison of it's map and the actual instance and finds a difference in the `Url` property. It will then execute a update statement against the underlying persistence store to record the change.

## EF Core Proxies

While not the recommended approach any more, EF Core also supports a different change tracking capability out of the box. This implementation uses proxying of the users classes in order to inject property change notifications in to the setters of objects. This has been deprecated due to performance issues (and no doubt the annoying requirement of marking all your properties virtual).

## Dashing

(Dashing)[https://github.com/Polylytics/dashing] is an ORM mostly written by me. In order to achieve change tracking in entities we take a drastically different approach. At compile time, just after the assembly is created, we modify the IL code of each domain class so that change tracking simply becomes part of the property setter.

In practice, you write the following code:

```
public class Blog {
    public int BlogId { get; set; }

    public string Name { get; set; }
}
```

but the build process re-writes this to be (along the lines of):

```
public class Blog {
    public int BlogId { get; set; }

    private string name;

    private string name_OldValue;

    private bool name_IsDirty;

    public string Name { 
        get {
            return this.name;
        } 
        set {
            if (!name_IsDirty && ((name == null && value != null) || (name != null && !name.Equals(value))))
			{
				name_OldValue = name;
				name_IsDirty = true;
			}

            name = value;
        }
    }
}
```

This has several advantages:

1. As a developer, you do not have to mark all your properties as `virtual`.
2. There is no need to copy all of the original values when loading an entity so we get a performance improvement as well as a memory reduction (over EF).
3. The entity knows itself whether it is dirty or not.

## Marten

(Marten)[https://github.com/JasperFx/marten] is a .NET Transactional Document DB and Event Store on PostgreSQL. Marten provides a range of options in terms of persisting changes to entities. Essentially, either you have automatic change tracking or you don't. However, having looked at the implementation you may want, at least for now, to stay away from the automatic change tracker. It's a very simple implementation, and probably works very well, but performance may vary.

Similarly to EF Core, Marten stores the intial values of a entity during load from PostgreSQL. And similarly to EF Core, Marten then compares the initial values against the loaded instances. However, it does this through serialization of the entities in to JSON and then comparison of those JSON strings to check for changes.

## Round up

So, it looks like we have 4 ways of tracking changes to entities:

1. Store the intial state and compare that to instances later.

    1a. Using a property by property mechanism

    1b. Using some other mechanism
2. Use runtime proxying to record data changes.
3. Modify compiled IL through some AOP technique to record data changes.

