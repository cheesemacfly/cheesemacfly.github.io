---
layout: post
title:  ".NET Core cookie authentication middleware and load balancing"
date:   2017-01-11 00:00:00
categories: blog
tags: dotnet .NET Core cookie middleware data protection key repository load balanced
---

Using the [.NET Core cookie authentication middleware](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie) as well as a load balanced environement usually makes the user's authentication "reset" between server hits.
That's because the data in the cookie is encoded using a key that [might only available in the server's memory](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings#data-protection-default-settings) (typically in a unix environement it will be the case).
So when the first server encryptes the cookie to send it back to the client it will use a key that it is not shared and other servers will not be able to decrypt the cookie on subsequent calls.

You could solve this by using sticky sessions on the load balancer or by moving this key to a file system location using:

{% highlight C# %}
public void ConfigureServices(IServiceCollection services)
{
   services.AddDataProtection()
       .PersistKeysToFileSystem(new DirectoryInfo(@"\\server\share\directory\"));
}
{% endhighlight %}

But sometimes you cannot share a directory between your web servers or you just want to use SQL to store the encryption key.
And that's exactely what we are going to do.

But before going further there is one **very important** point that needs to be [quoted from the documentation](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers):

> If you specify an explicit key persistence location, the data protection system will deregister the default key encryption at rest mechanism that the heuristic provided, so keys will no longer be encrypted at rest. It is recommended that you additionally specify an explicit key encryption mechanism for production applications.

And since that's what we're doing here, your keys are going to be stored not encrypted in your database. More information on that subject here: <https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-encryption-at-rest>.

With that out of the way, let's start!

## Prepare the storage

First, we need to create a table in our database that will hold the keys for our app:

{% highlight SQL %}
CREATE TABLE [dbo].[DataProtectionKeys](
	[FriendlyName] [nvarchar](max) NOT NULL,
	[XmlData] [nvarchar](max) NULL
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
{% endhighlight %}

The `FriendlyName` field is the actual id of the key and the `XmlData` will contain the XML data of the key deserialized in a string.

We can now create the model matching this table in our project:

{% highlight C# %}
public class DataProtectionKey
{
    [Key]
    public string FriendlyName { get; set; }
    public string XmlData { get; set; }
}
{% endhighlight %}

And add it to the context:

{% highlight C# %}
public class DBContext : DbContext
{
    public DBContext(DbContextOptions<DBContext> options) : base(options) { }

    public DbSet<DataProtectionKey> DataProtectionKeys { get; set; }
}
{% endhighlight %}

## Creating the repository

We can now write the repository (we must implement [IXmlRepository](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/extensibility/key-management#ixmlrepository)).

That's the most important part since that's where the write/read actions for the keys are going to happen.

{% highlight C# %}
public class DataProtectionKeyRepository : IXmlRepository
{
    private readonly DBContext _db;

    public DataProtectionKeyRepository(DBContext db)
    {
        _db = db;
    }

    public IReadOnlyCollection<XElement> GetAllElements()
    {
        return new ReadOnlyCollection<XElement>(_db.DataProtectionKeys.Select(k => XElement.Parse(k.XmlData)).ToList());
    }

    public void StoreElement(XElement element, string friendlyName)
    {
        var entity = _db.DataProtectionKeys.SingleOrDefault(k => k.FriendlyName == friendlyName);
        if (null != entity)
        {
            entity.XmlData = element.ToString();
            _db.DataProtectionKeys.Update(entity);
        }
        else
        {
            _db.DataProtectionKeys.Add(new DataProtectionKey
            {
                FriendlyName = friendlyName,
                XmlData = element.ToString()
            });
        }

        _db.SaveChanges();
    }
}
{% endhighlight %}

## Configure the app to use the new repository

Now we can simply use dependency injection to setup the new repository (in the Startup.cs file):

{% highlight C# %}
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    // custom entity framework key repository
    services.AddSingleton<IXmlRepository, DataProtectionKeyRepository>();
}
{% endhighlight %}

We are using Entity Framework to persist/retreive the data because it's fast to setup and easy to use but you can replace the data access layer by anything else.

There's also no exception handling or logging but that'd be a good improvement in a real world project!