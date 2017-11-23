---
layout: post
title: "ASP.NET Core MVC and Service Fabric"
date: 2017-11-22 00:00:00
categories: blog
tags: stateless service fabric dotnet ASP.NET Core MVC website
---

Hosting a website in service fabric has many advantages but it comes with a cost: you need to build your code around it which ties your project to the hosting plateform.

Once your project depends on the Service Fabric sdk, you will need a local cluster even just to debug the project. 
This is less than ideal, specialy when your frontend developers need to make changes and they run on a system that doesn't work with Service Fabric...
And having your project depend on the hosting plateform becomes a problem if you have to switch to a different one.

Starting with the version 6 of the sdk, you can create a Service Fabric project that runs exclusively on .net core 2.0
It will help a lot to solve the issue but it doesn't remove the requirement for a local cluster to run the project on your machine.

We are going to solve this issue by separating the hosting from the actual 'real' project.

Let's start by creating a ASP.NET Core MVC project using the version 2.0 of the framework:

![solution creation step 1](/assets/images/service-fabric-web-application/solution_creation_step1.png)

![solution creation step 2](/assets/images/service-fabric-web-application/solution_creation_step2.png)

If you start the app you should get the default standard template:

![web app running](/assets/images/service-fabric-web-application/web_app_running.png)

The idea here, is that we will be using this project as a regular ASP.NET Core MVC project.
So that when someone without the Service Fabric SDK wants to run the project, they can.
That's where all our 'real' code will live.
It doesn't mean that all your business logic must be in this project, it only means that this will be the main project for the website.

We can now create a folder `Hosting` in our solution that will contain all the hosting related projects:

![hosting folder](/assets/images/service-fabric-web-application/hosting_folder.png)

Now is time to add the Service Fabric project. We will choose a Stateless Service:

![service fabric app creation step 1](/assets/images/service-fabric-web-application/sf_app_creation_step1.png)

![service fabric app creation step 2](/assets/images/service-fabric-web-application/sf_app_creation_step2.png)

We can choose an empty template since it will ve replaced by our `MyWebApplication` anyway:

![service fabric app creation step 3](/assets/images/service-fabric-web-application/sf_app_creation_step3.png)

Now if you run the `MyWebApplication.ServiceFabric` in debug, you should see the "Hello World" defaut app we just created:

![service fabric hello world running](/assets/images/service-fabric-web-application/sf_hello_world_running.png)

As a side note, you could definitely use a Guest Executable (even using a `.cmd` to avoid having to compile against a plateform) and be pretty much done.

But this comes with a few disadvantages versus using a stateless service (not an exhaustive list):

* You will need to publish the web project before publishing the service fabric one (one more step)
* Debugging once running in service fabric will be more complex
* Using the configuration from Service Fabric would not be as easy

We will now add the reference to the 'real' project and use its `Startup` class.

Right click on the `MyService` -> `Dependencies` -> `Add Reference...`:

![adding reference](/assets/images/service-fabric-web-application/adding_reference.png)

A little bit of cleaning will make everything easier to read and maintain.
You can remove the `Startup.cs` file and the `wwwroot` folder from the `MyService` project.

At this point, if you run `MyWebApplication` it will still work of course.
But if you want to run it through Service Fabric...you will need to follow a few more steps.

First, we have to make sure that the `Views` and `wwwroot` from `MyWebApplication` folders make their ways into the `MyService` project.

We will be using `Link` content to avoid having multiple copies of the same files/folders:

{% highlight XML %}
<ItemGroup>
  <Content Include="..\MyWebApplication\Views\**\*.*" Link="Views\%(RecursiveDir)%(Filename)%(Extension)">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
  <Content Include="..\MyWebApplication\wwwroot\**\*.*" Link="wwwroot\%(RecursiveDir)%(Filename)%(Extension)">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
</ItemGroup>
{% endhighlight %}

You can definitely add more than just this two folders and even files (for example the `appsettings.json` could be a good candidate if you don't use the Service Fabric settings).

We also need to update the the folder where the web host is going to be looking for the content (this is only if you want/need debug for the Service Fabric project) in the `MyService.cs` file:

{% highlight C# %}
return new WebHostBuilder()
            .UseKestrel()
            .ConfigureServices(services => services.AddSingleton(serviceContext))
            // this is where you have to update the content folder
            // if you want/need debug for Service Fabric
            .UseContentRoot(AppDomain.CurrentDomain.BaseDirectory)
            .UseStartup<Startup>()
            .UseServiceFabricIntegration(listener, ServiceFabricIntegrationOptions.None)
            .UseUrls(url)
            .Build();
{% endhighlight %}

One last step that will be required, is to make sure that the file containing the depedencies to your `MyWebApplication` project is copied over to the other projects.
This is the `deps.json` file, and as far as I know, this is currently a [bug](https://github.com/NuGet/Home/issues/4412) and these steps will probably not be needed once it's fixed.

Edit the `MyService.csproj` file and add this section:

{% highlight XML %}
<!--
  Work around https://github.com/NuGet/Home/issues/4412. MVC uses DependencyContext.Load() which looks next to a .dll
  for a .deps.json. Information isn't available elsewhere. Need the .deps.json file for all web site applications.
-->
<Target Name="PostBuild" AfterTargets="PostBuildEvent">
  <ItemGroup>
    <DepsFilePaths Include="$([System.IO.Path]::ChangeExtension('%(_ResolvedProjectReferencePaths.FullPath)', '.deps.json'))" />
  </ItemGroup>
  <Copy SourceFiles="%(DepsFilePaths.FullPath)" DestinationFolder="$(OutputPath)" Condition="Exists('%(DepsFilePaths.FullPath)')" />
</Target>
{% endhighlight %}

You'll also need this section in the `MyWebApplication.ServiceFabric.sfproj`:

{% highlight XML %}
<!--
  Work around https://github.com/NuGet/Home/issues/4412. MVC uses DependencyContext.Load() which looks next to a .dll
  for a .deps.json. Information isn't available elsewhere. Need the .deps.json file for all web site applications.
-->
<Target Name="AfterPackage" AfterTargets="Package">
  <ItemGroup>
    <DepsFilePaths Include="$([System.IO.Path]::GetDirectoryName('%(_ResolvedProjectReferencePaths.FullPath)'))\*.deps.json" />
  </ItemGroup>
  <Copy SourceFiles="@(DepsFilePaths)" DestinationFolder="$(PackageLocation)\MyServicePkg\Code" />
</Target>
{% endhighlight %}

Note that in the second file, you need to specify the `DestinationFolder` with the name of the project we created earlier.

And that's it, you should now be able to run your project in Service Fabric both in debug mode and published:

![service fabric explorer](/assets/images/service-fabric-web-application/sf_explorer.png)

![service fabric web application running](/assets/images/service-fabric-web-application/sf_web_app_running.png)

Of course this will work for a local cluster as well as a hosted one.

So here you have it:

* `MyWebApplication` contains all your important code and can run anywhere .NET Core 2.0 is available
* `MyService` is the stateless service application referencing `MyWebApplication`
* `MyWebApplication.ServiceFabric` contains the artifacts for publishing the `MyService` project to a Service Fabric cluster

You can find the source code of the solution here: <https://github.com/cheesemacfly/ServiceFabricWebApplication>

I hope this makes sense and that it will be useful for people who want to use Service Fabric but do not want to commit 100% to it.