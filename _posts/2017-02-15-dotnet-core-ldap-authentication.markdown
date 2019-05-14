---
layout: post
title:  ".NET Core LDAP authentication"
date:   2017-02-15 00:00:00
categories: blog
tags: dotnet .NET Core LDAP authentication cookie authentication
---

.NET Core unfortunately doesn't [yet](https://github.com/dotnet/corefx/issues/2089) come with a native LDAP implementation...but you can use a third party library that will do the job for you: <https://github.com/dsbenghe/Novell.Directory.Ldap.NETStandard>

We will see how to integrate this third party library with a .NET Core website to authenticate the users with the cookie middleware and make the website only accessible to authenticated users members of the "Admins" group.

We are going to first create a very simple user class and an interface for the authentication service:

{% highlight C# %}
public class AppUser
{
    public string DisplayName { get; set; }
    public string Username { get; set; }
    public bool IsAdmin { get; set; }
}
{% endhighlight %}

{% highlight C# %}
public interface IAuthenticationService
{
    AppUser Login(string username, string password);
}
{% endhighlight %}

The goal is to be able to use the `IAuthenticationService` later in our DI container and inject our LDAP implementation.  
We will also assume that you have a configuration object defined as follow (example of the corresponding json format at the end):

{% highlight C# %}
public class LdapConfig
{
    public string Url { get; set; }
    public string BindDn { get; set; }
    public string BindCredentials { get; set; }
    public string SearchBase { get; set; }
    public string SearchFilter { get; set; }
    public string AdminCn { get; set; }
}
{% endhighlight %}

If you need more information about the configuration object I would recommend this part of the official documentation: <https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration#using-options-and-configuration-objects>

Next we create the LDAP implementation for this service inspired by [this example from the repository](https://github.com/dsbenghe/Novell.Directory.Ldap.NETStandard/blob/master/original_samples/Samples/VerifyPassword.cs):

{% highlight C# %}
class LdapAuthenticationService : IAuthenticationService
{
    private const string MemberOfAttribute = "memberOf";
    private const string DisplayNameAttribute = "displayName";
    private const string SAMAccountNameAttribute = "sAMAccountName";

    private readonly LdapConfig _config;
    private readonly LdapConnection _connection;

    public LdapAuthenticationService(IOptions<LdapConfig> config)
    {
        _config = config.Value;
        _connection = new LdapConnection
        {
            SecureSocketLayer = true
        };
    }

    public AppUser Login(string username, string password)
    {
        _connection.Connect(_config.Url, LdapConnection.DEFAULT_SSL_PORT);
        _connection.Bind(_config.BindDn, _config.BindCredentials);

        var searchFilter = string.Format(_config.SearchFilter, username);
        var result = _connection.Search(
            _config.SearchBase,
            LdapConnection.SCOPE_SUB,
            searchFilter,
            new[] { MemberOfAttribute, DisplayNameAttribute, SAMAccountNameAttribute },
            false
        );

        try
        {
            var user = result.next();
            if (user != null)
            {
                _connection.Bind(user.DN, password);
                if (_connection.Bound)
                {
                    return new AppUser
                    {
                        DisplayName = user.getAttribute(DisplayNameAttribute).StringValue,
                        Username = user.getAttribute(SAMAccountNameAttribute).StringValue,
                        IsAdmin = user.getAttribute(MemberOfAttribute).StringValueArray.Contains(_config.AdminCn)
                    };
                }
            }
        }
        catch
        {
            throw new Exception("Login failed.");
        }
        _connection.Disconnect();
        return null;
    }


}
{% endhighlight %}

We now add the configuration for LDAP and the `LdapAuthenticationService` object in the DI container:

{% highlight C# %}
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<LdapConfig>(Configuration.GetSection("ldap"));
        services.AddScoped<IAuthenticationService, LdapAuthenticationService>();
        
        // ...
    }
}
{% endhighlight %}

The controler handling the user authentication will be containing 2 routes: one for login and one for logout. The view model used in this example should contain 2 fields: `Username` and `Password`.  
(Make sure to include the `AllowAnonymous` attribute because later we will apply a default filter that will require authentication on all requests)

{% highlight C# %}
public class AccountController : Controller
{
    private readonly IAuthenticationService _authService;
    public AccountController(IAuthenticationService authService)
    {
        _authService = authService;
    }
    
    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> Login(LoginViewModel model)
    {
        if (ModelState.IsValid)
        {
            try
            {
                var user = _authService.Login(model.Username, model.Password);
                if (null != user)
                {
                    var userClaims = new List<Claim>
                    {
                        new Claim("displayName", user.DisplayName),
                        new Claim("username", user.Username)
                    };
                    if (user.IsAdmin)
                    {
                        userClaims.Add(new Claim(ClaimTypes.Role, "Admins"));
                    }
                    var principal = new ClaimsPrincipal(new ClaimsIdentity(userClaims, _authService.GetType().Name));
                    await HttpContext.Authentication.SignInAsync("app", principal);
                    return Redirect("/");
                }
            }
            catch (Exception ex)
            {
                ModelState.AddModelError(string.Empty, ex.Message);
            }
        }
        return View(model);
    }
    
    [Authorize(Roles = UserRoles.Everyone)]
    public async Task<IActionResult> Logout()
    {
        await HttpContext.Authentication.SignOutAsync("app");
        return Redirect("/");
    }
}
{% endhighlight %}

To setup the cookie authentication we call the [`UseCookieAuthentication`](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie#adding-and-configuring) method on the `IApplicationBuilder` object. We have to define our `/login` and `/logout` routes as well:

{% highlight C# %}
public class Startup
{
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            Events = new CookieAuthenticationEvents
            {
                // You will need this only if you use Ajax calls with a library not compatible with IsAjaxRequest
                // More info here: https://github.com/aspnet/Security/issues/1056
                OnRedirectToAccessDenied = context =>
                {
                    context.Response.StatusCode = (int)HttpStatusCode.Forbidden;
                    return TaskCache.CompletedTask;
                }
            },
            AuthenticationScheme = "app",
            LoginPath = new PathString("/login"),
            AutomaticAuthenticate = true,
            AutomaticChallenge = true
        });

        app.UseMvc(routes =>
        {
            routes.MapRoute(
                name: "login",
                template: "login",
                defaults: new { controller = "Account", action = "Login" }
            );
            routes.MapRoute(
                name: "logout",
                template: "logout",
                defaults: new { controller = "Account", action = "Logout" }
            );
        });
    }
}
{% endhighlight %}

And finally we can use a default filter applied to all routes that will require the users to be authenticated and in the "Admins" group (unless we specify the `AllowAnonymous` attribute on the controller/action):

{% highlight C# %}
public class ApplyPolicyOrAuthorizeFilter : AuthorizeFilter
{
    public ApplyPolicyOrAuthorizeFilter(AuthorizationPolicy policy) : base(policy) { }

    public ApplyPolicyOrAuthorizeFilter(IAuthorizationPolicyProvider policyProvider, IEnumerable<IAuthorizeData> authorizeData)
        : base(policyProvider, authorizeData) { }

    public override Task OnAuthorizationAsync(AuthorizationFilterContext context)
    {
        if (context.Filters.Any(f =>
        {
            var filter = f as AuthorizeFilter;
            //There's 2 default Authorize filter in the context for some reason...so we need to filter out the empty ones
            return filter?.AuthorizeData != null && filter.AuthorizeData.Any() && f != this;
        }))
        {
            return TaskCache.CompletedTask;
        }
        return base.OnAuthorizationAsync(context);
    }
}

// and in the Startup.cs:
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // default access requires admin access
        var isAdminUserPolicy = new AuthorizationPolicyBuilder().RequireRole("Admin").Build();
        services.AddMvc(options =>
        {
            options.Filters.Add(new ApplyPolicyOrAuthorizeFilter(isAdminUserPolicy));
        });
    }
}

{% endhighlight %}

Lastly here's a sample of what the LDAP section of your config file could look like:

{% highlight json %}
 "ldap": {
    "url": "ldap.local",
    "bindDn": "CN=user,OU=branch,DC=contoso,DC=local",
    "bindCredentials": "secret_password",
    "searchBase": "DC=contoso,DC=local",
    "searchFilter": "(&(objectClass=user)(objectClass=person)(sAMAccountName={0}))",
    "adminCn": "CN=Admins,OU=branch,DC=contoso,DC=local"
}
{% endhighlight %}