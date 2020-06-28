title: Explore Shadow Properties in EF Core
date: June 28, 2020
category: efcore
tags:
  - EF Core
  - database
  - data access
---

In the [previous post](https://shahabganji.me/efcore/explore-query-filters-ef-core/) we discovered Query Filters, a feature for EF Core 2.0 and later. Now I want to take a look at another popular feature [Shadow Properties](https://docs.microsoft.com/en-us/ef/core/modeling/shadow-properties). When designing applications we tend to keep our code clean and simple, however there are times that you need to add properties other than what is required in your main business use cases, `CreatedOn` and `LastUpdatedOn` are such well-known properties. Our code will look like the next:

```cs
public class Author
{
  // omitted properties

  public DateTimeOffset CreatedOn { get; set; }
  public DateTimeOffset? LastUpdatedOn { get; set; }
}
```

This is no good since we are eager to have our domain models reflect business models, this is where EF Core shadow properties come handy. Configuring shadow properties is so simple, in the `OnModelCreating` method of your DbContext( alternatively you could use `IEntityTypeConfiguration<T>` ):

```cs
builder.Property<DateTimeOffset>("CreatedOn");
builder.Property<DateTimeOffset>("LastUpdatedOn");
```

and then remove the actual properties from the underlying class. To access and set the values of shadow properties you could use the change tracker or static method `Property` of `EF` class.

```cs
context.Entry(author).Property("CreatedOn").CurrentValue = DateTime.Now;

// or

var authors = context.Authors
    .OrderBy(
        a => EF.Property<DateTime>(a, "LastUpdatedOn"));
```

Now we need a mechanism to tell the DbContext to automatically set values of these properties based on the state of the entity, but before that let me say that Shadow Properties come with a limitation, _they are accessible only through the **Change Tracker Api**, that means if they are loaded out side the DbContext or with `AsNoTracking` in DbContext, the values of these properties are not accessible_.

Well, so far so good, lets make our DbContext more smart with these values so that we don't need to be worry about when and how to set these values when coding our business rules in the application.

That, could be achieved by overriding the `SaveChanges` and `SaveChangesAsync` methods of DbContext and set these properties for the desired entities. To do so, I create an interface, `IHaveTrackingDates` but unlike `ICanBeSofDeleted` this is just a flag interface.

```cs
public interface IHaveTrackingDates { }

public class Author : ICanBeSoftDeleted , IHaveTrackingDates
{
 // omitted code
}
```

and then call the following method in the `OnModelCreating` method of the DbContext and remove the previous config.

```cs
public static void AddTrackingDatesShadowProperties(this ModelBuilder modelBuilder)
{
    var trackingDatesEntities =
                typeof(IHaveTrackingDates)
                .Assembly.GetTypes()
                    .Where(
                        type => typeof(IHaveTrackingDates)
                                    .IsAssignableFrom(type)
                                && type.IsClass
                                && !type.IsAbstract);

    foreach (var entity in trackingDatesEntities)
    {
        modelBuilder.Entity(entity)
            .Property<DateTimeOffset>("CreatedOn")
            .IsRequired();
        modelBuilder.Entity(entity)
            .Property<DateTimeOffset?>("LastUpdatedOn")
            .IsRequired(false)
            .HasColumnName("LastUpdatedAt");
    }
}
```

Last make your DbContext smart enough by telling it how to detect these entities and set their properties, call this method in the overridden SaveChanges methods.

```cs
private void UpdateTrackingDates()
{
    var trackingDatesChangedEntries = this.ChangeTracker.Entries()
        .Where(entry => entry.State == EntityState.Added
                    || entry.State == EntityState.Modified
            && entry.Entity is IHaveTrackingDates
        );

    foreach (var entry in trackingDatesChangedEntries)
    {
        if (entry.State == EntityState.Modified)
            this.Entry(entry).Property("LastUpdatedOn")
                .CurrentValue = DateTimeOffset.UtcNow;
        else if (entry.State == EntityState.Added)
            this.Entry(entry).Property("CreatedOn")
                .CurrentValue = DateTimeOffset.UtcNow;
    }
}
```

```cs
public override int SaveChanges()
{
    this.UpdateTrackingDates();
    return base.SaveChanges();
}

public override Task<int> SaveChangesAsync(
    CancellationToken cancellationToken = new CancellationToken())
{
    this.UpdateTrackingDates();
    return base.SaveChangesAsync(cancellationToken);
}
```

# Closing

Shadow properties are giving us an opportunity to clean up our domain models from codes that are not necessarily based on the business rules, as other features and technologies they have their own pros and cons we might consider when using them.

You could find the complete code of this post on [github](). Now that we know how to handle shadow properties and also able to add global query filters it would be a good practice to change the soft delete mechanism as a shadow property as well. Have a great day and enjoy coding!
