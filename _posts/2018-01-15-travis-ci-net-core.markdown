---
layout: post
title:  "Setup Travis CI to compile a .NET CORE project, run the tests and package it for NuGet"
date:   2018-01-15 00:00:00
categories: blog
tags: C# .NET Core Travis AppVeyor CI xUnit NuGet
---

[3 years ago](/blog/2015/01/18/travis-ci-csharp.html), I wrote a short post about setting up Travis CI with .Net 4.5 (including running the tests with NUnit) and now is time for an update.  

I have upgraded the library in the example ([MTAServiceStatus on Github](https://github.com/cheesemacfly/MTAServiceStatus)) and it is now targeting .NET Standard.
The test framework this time around is [xUnit](https://xunit.github.io/) ([MSTest is still not supported with Travis](https://docs.travis-ci.com/user/languages/csharp/#Other-test-frameworks)).

The Travis CI configuration file is now pretty straigthforward and looks like this:

{% highlight yaml linenos=table %}
language: csharp
mono: none
dotnet: 2.0.0

install:
- dotnet restore

script:
 - dotnet build
 - dotnet test MTAServiceStatus.Tests/MTAServiceStatus.Tests.csproj
{% endhighlight %}

The only part that might requires an explanation is `mono: none`.
By default Travis uses the latest mono release but we do not need it anymore with .NET Core.
Setting `mono: none` tells Travis CI to disable it and on the next line we can set the version of the .NET Core SDK that we want to use: `dotnet: 2.0.0`.
All the information about this requirement is here: <https://docs.travis-ci.com/user/languages/csharp/#.NET-Core>.

Since setting up Travis CI was easier than last time, I took the opportunity to setup AppVeyor to package the library and upload it to NuGet.
It will also create a release in GitHub with a zip archive of all the build files and a copy of the NuGet package.
This is a little bit more involved than setting up only the Travis CI process but you can find all the information below:

{% highlight yaml linenos=table %}
version: 1.0.{build}
skip_tags: true
image: Visual Studio 2017
configuration: Release
dotnet_csproj:
  patch: true
  file: '**\*.csproj'
  version: '{version}'
  package_version: '{version}'
  assembly_version: '{version}'
  file_version: '{version}'
  informational_version: '{version}'
before_build:
- cmd: nuget restore
build:
  publish_nuget: true
  verbosity: minimal
deploy:
- provider: NuGet
  api_key:
    secure: buTns+OxOaxT6NLivjBNcBXweOlxv9y512TP605PAE0OQRhUzjv4B99Suga/R5Eq
  on:
    branch: master
- provider: GitHub
  tag: v$(appveyor_build_version)
  release: v$(appveyor_build_version)
  auth_token:
    secure: ocqFamtfZzSNeNzuwUODEg8Xvjtgsui2P1ban6Gm4UMBpisfJkqVYScfYb7UPRCC
  artifact: /.*\.nupkg/
  on:
    branch: master
{% endhighlight %}

I'm using the version number from AppVeyor which is increased every time a build is triggered.
You'll find all the information about the settings here: <https://www.appveyor.com/docs/>.  

I also had to switch from setting the parameters up in the web interface to using a yaml configuration file to get everything working.