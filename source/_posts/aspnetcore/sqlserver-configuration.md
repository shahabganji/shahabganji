title: Configuration settings from Sql Server in ASP.NET Core Applications
date: October 25 2020
category: aspnetcore
tags:
    - aspnetcore
    - configuration
    - sqlserver
---

A friend of mine has a requirement in which he was interested to put the configuration settings of an ASP.NET Core application inside a SQL Server table and read them at startup of the application with a mechanism similar to what we already have for json files; also he wanted to have the ability that these variable being overridden or override other settings if applicable. In this article I am going to describe how that could be achievable and how to implement such feature.

<!-- more -->

Creating such mechanism is two fold, first you need to create a configuration source and then a configuration provider. The former implements `IConfigurationSource` interface, and can hold the settings of your configuration and it should implement the `Build` method, which returns an `IConfigurationProvider` instance that the `IConfigurationBuilder` can leverage to build the configurations of the application. Let's see them in action:


```cs
 public class SqlServerConfigurationSource : IConfigurationSource
{
    public string Server { get; set; } = "localhost";
    public string Database { get; set; } = "ConfigurationDb";
    public string User { get; set; } = "sa";
    public string Password { get; set; } = "yourStrong(!)Password";
    public string Table { get; set; } = "dbo.Configuration";
    public string KeyColumn { get; set; } = "Key";
    public string ValueColumn { get; set; } = "Value";

    public IConfigurationProvider Build(IConfigurationBuilder builder)
            => new SqlServerConfigurationProvider(this);
}
```

As you can see, it only has some properties which are holding the required values for the concrete `IConfigurationProvider`, and in the `Build` method it simply returns an instance of the concrete class that inherits from the `ConfigurationProvider`.

```cs
public class SqlServerConfigurationProvider : ConfigurationProvider , IDisposable
{
    private readonly SqlServerConfigurationSource _configurationSource;

    public SqlServerConfigurationProvider(SqlServerConfigurationSource configurationSource)
    {
        _configurationSource = configurationSource;
    }

    public override void Load()
    {
        // here you should connect to sql server and fetch the configurations
    }

    public void Dispose()
    {
        // cleanup all the resources
    }
}
```

In the concrete configuration provider you should override the `Load` method and fetch the configurations from the external storage, in this case, SQL Server; but as you see it can simply be any kind of storage.

Easy!? you're right; the next step could be to add some extension methods to facilitate the usage of these two classes so the developers could call `AddSqlServerDb` in the `ConfigureAppConfiguration` in the `Program.cs`, for instance, to be able to load the configuration from sql server.

```cs
 public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration(configure =>
        {
            configure.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
            configure.AddSqlServerDb();
        })
        .ConfigureWebHostDefaults(webBuilder => { webBuilder.UseStartup<Startup>(); });
```

With the above-mentioned configuration settings if there is an intersection between keys defined in the `appsettings.json` file and the `SqlServer` those in the SQL Server will override it.

That's it. you could take advantage of these two primary classes to bring flexibility to your applications and read the required configuration from any external storage. Find the source code for this article on [this](https://github.com/shahabisblogging/SQLServerConfigurationSample) GitHub repository.

Have a great day! and enjoy coding ;)