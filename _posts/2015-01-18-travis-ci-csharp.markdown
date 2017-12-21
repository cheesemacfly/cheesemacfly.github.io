---
layout: post
title:  "Setup Travis CI to compile a C# project and run NUnit tests"
date:   2015-01-18 23:00:00
categories: blog
tags: C# .NET Travis CI NUnit
---
I recently built a C# [library] to access the New York City MTA service status feed.  
This project includes unit tests built with the [NUnit] framework and it was my first shot at using Tavis CI.  

It is fairly easy to find examples of the `.travis.yml` file including the `apt-get` commands installing mono but it isn't required anymore.  
Travis CI has now an easier way of doing things when it comes to building C# (even if it is stated that this is [beta][doc] and that it could be removed at anytime...)

Here is what the `.travis.yml` file I used ended up looking:

{% highlight yaml linenos=table %}
language: csharp
solution: solution-name.sln
before_install:
  - sudo apt-get install nunit-console
before_script:
  - nuget restore solution-name.sln
after_script:
  - nunit-console Tests/bin/Release/Tests.dll
{% endhighlight %}

The first and second lines are obvious and [documented][doc].  
But using [NUnit] is not and requires the console to be installed (line 4) and run using the test project output (the .dll file, line 8).  
`Release` (which is the default) can be replaced by the build configuration you are using using the following `.travis.yml`file sample:

{% highlight yaml linenos=table %}
language: csharp
script:
  - xbuild /p:Configuration=Debug solution-name.sln
before_install:
  - sudo apt-get install nunit-console
before_script:
  - nuget restore solution-name.sln
after_script:
  - nunit-console Tests/bin/Debug/Tests.dll
{% endhighlight %}

Line 3 and 9 need to be adapted to your build configuration (`Debug` might need to be changed to match your configuration name).  

You can see a "real life" example here: [MTAServiceStatus on Github](https://github.com/cheesemacfly/MTAServiceStatus/tree/v0.3.0)  
Associated Travis CI badge: [![Build Status](https://api.travis-ci.org/cheesemacfly/MTAServiceStatus.svg?tag=v0.3.0)](https://travis-ci.org/cheesemacfly/MTAServiceStatus)  
Travis CI configuration used for this project: [`.travis.yml` file on master](https://github.com/cheesemacfly/MTAServiceStatus/blob/v0.3.0/.travis.yml)

[NUnit]:        http://www.nunit.org/
[library]:      https://github.com/cheesemacfly/MTAServiceStatus/tree/v0.3.0
[doc]:          http://docs.travis-ci.com/user/languages/csharp/
