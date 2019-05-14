---
title: IIS double authentication
date: 2015-05-31 00:00:00 Z
categories:
- blog
tags:
- IIS
- rewrite
- authentication
- ARR
- proxy
layout: post
---

Setting up double authentication for a website running on IIS 7+ running in "Integrated mode"  can be a very hard task.  
If you want, as an example, to have a "Basic authentication" to password protect your website and use the "Forms authentication" to allow your users to log in on this same website, it could take a lot of work.  
There's already a solution here: <http://mvolo.com/iis-70-twolevel-authentication-with-forms-authentication-and-windows-authentication/> but it requires code changes and add dependencies to your project.  

If you want to get this done by only tweaking up your server's configuration (to password protect a stage environment for a limited time for example) there is a solution.  

Before we go further, I want to be clear that this is *not a pretty solution* and you should not be using it on production. And even tho it doesn't require code changes, it will most probably require that you install some IIS modules.  

The general idea is that we will create a gate website to password protect the real application.  
All the requests will have to go through the gate and you'll be able to apply any authentication you want ("Basic authentication" in the following example).  
Once the user has successfully authenticate, we will rewrite every request to the real website.

It only takes 5 steps to get this done:

1.  Change the binding of your website to a port that is not being used on your server (for example 8888)  
![binding settings](/assets/images/iis-double-authentication/binding.png)
2.  Create a second website (we will call it `gate`) and setup the binding so that all the traffic of your real website is directed to this one. That's the website where you will want to setup your second authentication (usually the "Basic authentication")  
![gate setup](/assets/images/iis-double-authentication/gate.png)
3.  Install the "[URL Rewrite]" and "[Application Request Routing]" modules (using web platform installer makes it easy)
![wpi cart](/assets/images/iis-double-authentication/wpi_cart.png)
4.  Enable the proxy on the ARR module (this is the key part to be able to rewrite the content)

    - Open ARR from the root of the server and go into the proxy settings (from the right column in "Server Proxy Settings...")  
    ![open arr](/assets/images/iis-double-authentication/arr_open.png)

    - Enable the proxy option (you don't need to change the other parameters)  
    ![enable the proxy](/assets/images/iis-double-authentication/proxy_enabled.png)
5.  Add the following rewrite rule to the `gate` website (select the `gate` website, go into 'URL Rewrite' and select 'Add Rule(s)...')
![rewrite rule](/assets/images/iis-double-authentication/rewrite_rule.png)
This will produce the following XML code in the gate website's web.config:
{% highlight XML %}
<rule name="gate">
  <match url=".*" />
  <action type="Rewrite" url="http://localhost:8888/{R:0}" />
</rule>
{% endhighlight %}  

`http://localhost:8888` is your real website and `{R:0}` is the requested path. You can find more information about the rewrite module in the post [IIS Rewrite Redirect Back-references]({% post_url 2015-02-21-rewrite-redirect-iis %}).

This solution will get you up and running with 2 different authentications in no time and with very little requirements.  
You can now have a password protected dev environment without breaking your website authentication.

[Url Rewrite]: http://www.iis.net/downloads/microsoft/url-rewrite
[Application Request Routing]: http://www.iis.net/downloads/microsoft/application-request-routing
