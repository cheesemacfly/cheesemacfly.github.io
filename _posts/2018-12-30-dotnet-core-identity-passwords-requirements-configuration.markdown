---
layout: post
title:  "ASP.NET Identity passwords requirements configuration"
date:   2018-12-30 00:00:00
categories: blog
tags: C# dotnet core IPasswordValidator Identity Configuration
---

Using ASP.NET Core Identity is great out of the box to manage users in a web app.
The default configuration give pretty standard requirements for passwords when users are creating accounts.

The most simple way to change the settings can be found in the [documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity-configuration?view=aspnetcore-2.1#password)

{% highlight C# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddIdentity<ApplicationUser, IdentityRole>();
    
        services.Configure<IdentityOptions>(options =>
        {
            // Default Password settings.
            options.Password.RequireDigit = true;
            options.Password.RequireLowercase = true;
            options.Password.RequireNonAlphanumeric = true;
            options.Password.RequireUppercase = true;
            options.Password.RequiredLength = 6;
            options.Password.RequiredUniqueChars = 1;
        });
    }
}
{% endhighlight %}

This is great and will be enough in many cases.
You could also move the settings in a seperate method to avoid cluttering the `ConfigureServices` method.

{% highlight C# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
         services.AddIdentity<ApplicationUser, IdentityRole>(SetIdentityOptions);
    }

    private static void SetIdentityOptions(IdentityOptions options)
    {
        options.Password.RequireDigit = true;
        options.Password.RequireLowercase = true;
        options.Password.RequireNonAlphanumeric = true;
        options.Password.RequireUppercase = true;
        options.Password.RequiredLength = 6;
        options.Password.RequiredUniqueChars = 1;
    }
}
{% endhighlight %}

Now if we want to make something a bit more complex, for example making sure that the password doesn't contain the username, we'll have to implement the `IPasswordValidator` interface.

{% highlight C# %}
public class MyPasswordValidator : IPasswordValidator<ApplicationUser>
{
    public Task<IdentityResult> ValidateAsync(UserManager<ApplicationUser> manager, ApplicationUser user, string password)
    {
        return Task.FromResult(Validate(user, password));
    }

    private IdentityResult Validate(ApplicationUser user, string password)
    {
        if (password.Contains(user.UserName, System.StringComparison.OrdinalIgnoreCase))
        {
            return IdentityResult.Failed(new IdentityError
            {
                Code = "UserNameInPassword",
                Description = "Passwords cannot contain the account's username"
            });
        }

        return IdentityResult.Success;
    }
}
{% endhighlight %}

We also need to set it up in the `Startup.cs` to have it called on password validation.

{% highlight C# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
         services
            .AddIdentity<ApplicationUser, IdentityRole>()
            .AddPasswordValidator<MyPasswordValidator>();
    }
}
{% endhighlight %}

This works and is easy enough to put together but what if we want to have all of our password validation in one place to make maintenance easier?
For example, let's add the length validation to our `MyPasswordValidator` class.

{% highlight C# %}
public class MyPasswordValidator : IPasswordValidator<ApplicationUser>
{
    public Task<IdentityResult> ValidateAsync(UserManager<ApplicationUser> manager, ApplicationUser user, string password)
    {
        return Task.FromResult(Validate(user, password));
    }

    private IdentityResult Validate(ApplicationUser user, string password)
    {
        if (password.Length < 6)
        {
            return IdentityResult.Failed(new IdentityError
            {
                Code = "PasswordTooShort",
                Description = "Passwords must be at least six characters"
            });
        }

        if (password.Contains(user.UserName, System.StringComparison.OrdinalIgnoreCase))
        {
            return IdentityResult.Failed(new IdentityError
            {
                Code = "UserNameInPassword",
                Description = "Passwords cannot contain the account's username"
            });
        }

        return IdentityResult.Success;
    }
}
{% endhighlight %}

Now if you run this code and send the password `pass123`, you will get a validation error saying that you must provide a password with at least 8 characters.
This is because the default validator still runs when you use the `AddPasswordValidator` to set up your custom validator.

Let's find out why.

When we call `AddIdentity` in our Startup class, it is actually an extension method calling a bunch of other methods (as you can see here <https://github.com/aspnet/AspNetCore/blob/master/src/Identity/src/Identity/IdentityServiceCollectionExtensions.cs#L38>).
But the one that is of interest to us is [`services.TryAddScoped<IPasswordValidator<TUser>, PasswordValidator<TUser>>();`](https://github.com/aspnet/AspNetCore/blob/master/src/Identity/src/Identity/IdentityServiceCollectionExtensions.cs#L82)
This method is trying to register the default [`PasswordValidator`](https://github.com/aspnet/AspNetCore/blob/master/src/Identity/src/Core/PasswordValidator.cs) into the services if it isn't already present.

Knowing that, it's easy to skip the default validation. Registering the `IPasswordValidator` __before__ the `AddIdentity` method does it will do the trick.

{% highlight C# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.TryAddScoped<IPasswordValidator<ApplicationUser>, MyPasswordValidator>();
        
        // This line must come after
        services.AddIdentity<ApplicationUser, IdentityRole>();
    }
}
{% endhighlight %}

We have basically replaced `AddPasswordValidator` so we do not need it anymore. We actually could not use it for this purpose since it's an extension method for the [`IdentityBuilder`](https://github.com/aspnet/AspNetCore/blob/master/src/Identity/src/Core/IdentityBuilder.cs#L100) type.