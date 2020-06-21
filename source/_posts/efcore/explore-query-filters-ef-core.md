title: Explore Global Query Filters in EF Core
date: June 21, 2020
category: efcore
tags:
  - EF Core
  - database
  - data access
---

In this article we are going to check one of the features of Entity Framework Core, [Global Query Filters](https://docs.microsoft.com/en-us/ef/core/querying/filters); this was introduced first in EF Core 2.0 and is just a query predicate that will be appended to the end of where clause for queries on entities for which this feature has been activated. Some common scenarios would be `Soft delete` and `Multi tenancy`.

Consider that you are writing an application in which entities can be soft deleted, why not completely delete those entities? <!-- more --> [just don't do that](http://udidahan.com/2009/09/01/dont-delete-just-dont/), said Udi Dahan. Okay, let's create an `Author` entity:

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

If we want to add a query filter to the `Author` entity, that should be done either in `OnModelCreating` method of the `DbContext`, or in the `EntityTypeConfiguration<T>` class related to entity `T`. For now, we could create a `DbContext` and override its `OnModelCreating` method, then configure the `Author` entity by telling the context to automatically _**filter out** soft-deleted_ records, their `IsDeleted` property is `true`.

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

To achieve that, `HasQueryFilter` method is being used, it is defined on `EntityTypeBuilder` and takes  an `Expression` of `Func<Author, bool>` which means it will take an author instance and returns a boolean based on the conditions defined, in this case, the author not being (soft) deleted. Now, whenever you query the `Author` entity this query will also get appended to whatever you have provided for your where clause by the DbContext.

```cs
// omitted code
var authors = context.Authors
                .Where(author => author.LastName.StartsWith("joh"));
```
 resulting in a database query:

 ```sql
SELECT  a.*
FROM    dbo.Authors as a
WHERE   a.LastName  LIKE 'joh%'
    AND a.IsDelete  =     0
 ```

So far so good. This could be a common scenario for almost all of your entities(aggregate roots, if interested) and we are reluctant to repeat same code for every individual entity([DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)).

Let's first extract an interface, `ICanBeSoftDeleted`, which desired entities will implement that.

```cs
public interface ICanBeSoftDeleted
{
    bool IsDeleted{ get; set;}
}

public class Author : ICanBeSoftDeleted
{
    // omitted code
}

public class Book : ICanBeSoftDeleted
{
    // omitted code
}

```

Now we have to configure the common query filter for each entity, in the previous version we used a **generic** overload of `modelBuilder.Entity<T>` method, now it is not possible though, so we have to generate a _Lambda Expression_ for each entity to be able to use the non-generic overload:

```cs
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var softDeleteEntities = typeof(ICanBeSoftDeleted).Assembly.GetTypes()
                .Where(type => typeof(ICanBeSoftDeleted)
                                .IsAssignableFrom(type)
                                && type.IsClass
                                && !type.IsAbstract);

    foreach (var softDeleteEntity in softDeleteEntities)
    {
        modelBuilder.Entity(softDeleteEntity)
                    .HasQueryFilter(
                        GenerateQueryFilterLambda(softDeleteEntity)
                        );
    }

    base.OnModelCreating(modelBuilder);
}

```

let's discover the `GenerateQueryFilterLambda`, it takes the type of the entity and will return a LambdaExpression of Func<Type, bool>, we should generate `e => e.IsDeleted == false`, right? see how to generate each part using [Expressions](https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression?view=netcore-3.1) in .Net Core.

```cs
 private LambdaExpression GenerateQueryFilterLambda(Type type)
{
    // we should generate:  e => e.IsDeleted == false
    // or: e => !e.IsDeleted

    // e =>
    var parameter = Expression.Parameter(type, "e");

    // false
    var falseConstant = Expression.Constant(false);

    // e.IsDeleted
    var propertyAccess = Expression.PropertyOrField(parameter, nameof(ICanBeSoftDeleted.IsDeleted));

    // e.IsDeleted == false
    var equalExpression = Expression.Equal(propertyAccess, falseConstant);

    // e => e.IsDeleted == false
    var lambda = Expression.Lambda(equalExpression, parameter);

    return lambda;
}
```

Comments on top of each line indicating that what part of the (lambda) expression is going to be generated. This way we can simply implement the interface and our DbContext automatically detect and add the common global filter to our queries for the specified entities. At the end whenever you want to disable query filter use `IgnoreQueryFilters()` on your LINQ query.

```cs
var authors = context.Authors
                .Where(author => author.LastName.StartsWith("joh"))
                .IgnoreQueryFilters();
```

# Conclusion

Most of the times there are business query patterns that will apply globally on some entities in you application, bu employing `Query Filters` of EF Core you could simply and easily implement such requirement. There is also a limitation, these filters can only be applied to the root entity of an inheritance hierarchy. Finally, [here](https://github.com/shahabisblogging/SampleEfCoreBookStore) you could find a sample for this article. Have a great day and enjoy coding.
