title: Ensure Uniqueness of Nullable Columns
date: July 5, 2020 08:47 PM
category: database
tags:
  - sql server
  - database
---

A couple of weeks ago I encountered a tweet in which one of the questions expected from a candidates in a job interview was the reason to write this post, How do you ensure that duplicate keys are prevented on a column that allows `NULL` values?

That reminds me of the time that I was focused on databases and I had to handle such problem for one of our databases, here we are going to talk about two approaches to address this issue in SQL Server.

<!-- more -->

Consider you have a column such as `Employee` with a column `SSN` that can be null at some point and will be filled when any SSN number issued for that employee. The script for such table looks like the next:

```sql
CREATE  TABLE   dbo.Employee(
      Id          BIGINT          NOT NULL    IDENTITY
          CONSTRAINT PK_Employee_Id Primary KEY
  ,   FirstName   nVarChar(50)    NOT NULL
  ,   LastName    nVarChar(100)   NOT NULL
  ,   SSN         Char(10)            NULL
          CONSTRAINT UQ_Employee_SSN UNIQUE
)
ON [PRIMARY]
GO
```

In the preceding code snippet if I add a unique constraint on the `SSN` column, when I want to insert more than one employee with `NULL` values for the `SSN` column we will face an error indicating a `UNIQUE KEY` violation. Don't believe me, try it by yourself:

```sql
INSERT INTO dbo.Employee( FirstName , LastName , SSN )
    VALUES( 'Jane' , 'Doe' , NULL )

INSERT INTO dbo.Employee( FirstName , LastName , SSN )
    VALUES( 'John' , 'Doe' , NULL )
```

```
Msg 2627, Level 14, State 1, Line 1
Violation of UNIQUE KEY constraint 'UQ_Employee_SSN'. Cannot insert duplicate key in object 'dbo.Employee'. The duplicate key value is (<NULL>).
```

You see the problem in action, lets touch on the solution. There are some approaches to tackle this problem, of which I explain two of them.

# Computed Columns

[Computed columns](https://docs.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table?view=sql-server-ver15) are a special kind of columns in SQL Server that gives the ability to store values based on a formula, in this approach we could define a computed column which takes the unique column, in this example `SSN`, and another combination of unique columns, I'll take the primary key. so lets alter the previously defined table, first we remove the `Unique` constraint on the `SSN` column, and then define the computed column and add another unique constraint on that:

```sql
ALTER TABLE dbo.Employee
    DROP    Constraint UQ_Employee_SSN
GO
```

```sql
ALTER TABLE dbo.Employee
  ADD SSN_CMPTD AS
    CASE
      WHEN SSN IS NOT NULL
        THEN SSN
      ELSE
        CAST( Id AS Char(10) )
    END
GO

ALTER TABLE dbo.Employee
  ADD CONSTRAINT  UQ_Employee_SSN_CMPTD UNIQUE(SSN_CMPTD)
GO

```

Now try the above-mentioned `INSERT` commands, this time both will pass, and the next two inserts will also pass as they don't violate any constraint.

```sql
INSERT INTO dbo.Employee( FirstName , LastName , SSN )
    VALUES( 'Shahab' , 'Ganji' , '0987654321' )

INSERT INTO dbo.Employee( FirstName , LastName , SSN )
    VALUES( 'Some' , 'One' , '1234567890' )
GO
```

now if you try to insert a real duplicate key, you will face a _unique constraint violation_ error, but this time because of a real duplicate key, check the value at the end of the error and the column name which the violation occurred on.

```sql
INSERT INTO dbo.Employee( FirstName , LastName , SSN )
    VALUES( 'Invalid' , 'Operation' , '1234567890' )
```

```
Msg 2627, Level 14, State 1, Line 2
Violation of UNIQUE KEY constraint 'UQ_Employee_SSN_CMPTD'. Cannot insert duplicate key in object 'dbo.Employee'. The duplicate key value is (1234567890).
```

So far so good, right? Let's try the other approach, I preferred when designing such scenarios.

# Filtered Indexes

[Filtered Index](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes?view=sql-server-ver15) is a feature described on the SQL Server documentation site as  **_an optimized nonclustered index especially suited to cover queries that select from a well-defined subset of data_**. Crystal clear? okay, let's remove the computed column and its unique constraint defined earlier. (Alternatively, you could re-create the table without the unique constraint on the `SSN` column).

```sql
ALTER TABLE dbo.Employee
    DROP CONSTRAINT UQ_Employee_SSN_CMPTD
GO
ALTER TABLE dbo.Employee
    DROP   COLUMN SSN_CMPTD
GO
```

Now we could define the desired filtered index with the following script:

```sql
CREATE UNIQUE NONCLUSTERED INDEX Idx_Employee_SSN
    ON  dbo.Employee( SSN )
WHERE   SSN IS NOT NULL
GO
```

Awesome! you could try the previous insert commands to check against this new unique index.

# Conclusion

The latter approach have some advantages over the former, of which the most important one is that we could use this unique index as a [Covering Index](http://dbadiaries.com/sql-server-covering-index-and-key-lookup#:~:text=A%20covering%20index%20is%20a,it%20is%20a%20faster%20operation.) for queries on the `Employee` table to prevent a `Key Lookup`, that could be a boost in performance of queries relating that table. Moreover, the former approach occupies more space since it has to store values of the computed column somewhere on the disk; another obstacle for the former approach would be the complexity of matching (another) column(s) to satisfy the uniqueness of the computed columns in the absence of an SSN.

I hope you found this post fruitful, have a good evening and enjoy coding!
