title: Explore Shadow Properties in EF Core
date: June 23, 2020
category: efcore
tags:
  - EF Core
  - database
  - data access
---

# This is _**DRAFT**_

In the previous post we discovered Query Filters, a feature for EF Core 2.0 and later. Now I want to take a look at another popular feature [Shadow Properties](https://docs.microsoft.com/en-us/ef/core/modeling/shadow-properties). When designing applications we tend to keep our code clean and simple, howerever there are times that you need to add properties other than what is required in your main business use cases, `CreatedOn` and `UpdatedOn` are such well-known properties. Our code will look like the next:

```cs
public class Author
{
  // omitted properties
  
  public DateTimeOffset CreatedOn { get; set; }
  public DateTimeOffset? LastUpdatedOn { get; set; }
}
```

This is no good since we are eager to have our domain models reflect business models, this is where EF Core shadow properties come handy. COnfiguring shadow properties is so simple, in the `OnModelCreating` method of your DbContext( alternatively you could use `IEntityTypeConfiguration<T>` ):

```cs
builder.Property<DateTimeOffset>("CreatedOn");
builder.Property<DateTimeOffset>("LastUpdatedOn");
```

and then remove the actual properties from the underlying class. To access and set the values of shadow properties you could use the change tracker or static method `Property` of `EF` class.

```cs
context.Entry(author).Property("CreatedOn").CurrentValue = DateTime.Now;

// or 

var authors = context.Authors
    .OrderBy(a => EF.Property<DateTime>(a, "LastUpdatedOn"));
```



