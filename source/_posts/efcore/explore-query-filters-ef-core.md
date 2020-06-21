title: Explore Global Query Filters in EF Core
date: June 21, 2020
category: efcore
tags: 
  - EF Core
  - database
  - data access
---

# THIS IS DRAFT


In this article we are going to check one of the features of Entity Framework Core, Global Query Filters; this was introfuced first in EF Core 2.0 and is just a query predicate that will be appended to the where clause of the entitities that this feature is activated for them. Some common scenarios for such feature would be `Soft delete` and `Multi tenancy`.


Lets see that in action.

<!-- more -->

Consider that you are writing and application in which entities can be soft deleted, why not completely delete those entities? [jus don't do that](http://udidahan.com/2009/09/01/dont-delete-just-dont/), said Udi Dahan. Okay, let's create an `Author` entity:

```cs
public class Author
{
    // omitted code

    public Guid Id { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public DateTime DateOfBirth { get; private set; }
    public bool IsDeleted { get; private set; }
}
```

If we want to add a query filter to the `Author` entity, that should be done either in `OnModelCreating` method of the `DbContext`, or in the `EntityTypeConfiguration<T>` class relate to this entity. For now, we could create a `DbContext` and override its `OnModelCreating` method, then configure the `Author` entity by telling the context to automatically filter out those who are sof delete, their `IsDeleted` property is `false`.

```cs
public class BookstoreDbContext : DbContext
{
    // omitted code 

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Author>()
                    .HasQueryFilter(author => !author.IsDeleted);
        
        base.OnModelCreating(modelBuilder);
    }

    public DbSet<Author> Authors { get; set; }
}
```

To achieve that, `HasQueryFilter` method is being used, it is defined on `EntityTypeBuilder` and takes  an `Expression` of `Func<Author, bool>` which means it will take an author instance and returns a boolean based on the conditions defined, in this case, the author not being (soft) deleted. Now, whenever you query the `Author` entity this query will also get appended to whatever you have provided for your where clause by the dbcontext.

```cs
// omitted code
var authors = context.Authors
                        .Where(author => author.LastName.StartsWith("joh"));
```
 resulting in a database query like:

 ```sql
SELECT  a.*
FROM    dbo.Authors as a
WHERE   a.LastName  LIKE 'joh%' 
    AND a.IsDelete  =     0
 ```

So far so good. This could be a common scenario for almost all of your entities(aggregae roots, if inetersted) and we are reluctant to repeat same code for every individual entity([DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)).