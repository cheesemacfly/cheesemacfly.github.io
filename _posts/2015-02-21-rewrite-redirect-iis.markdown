---
layout: post
title:  "Understand IIS rewrite and redirect rules"
date:   2015-02-21 00:00:00
categories: blog
tags: IIS Rewrite Redirect Back-references
---
A while ago, I answered a question on [stackoverflow] asking clarifications about the IIS rewrite module and how back references work.
This question has received quite some attention since it was posted and I wanted to write a simple guide for this module here.  

## What is the IIS rewrite module?

The rewrite module for IIS can be used to execute 2 actions:

 - rewrite => the URL stays the same but the content is loaded from somewhere else
 - redirects => when the user's browser is taken to a new URL

Both use the same rules and conditions to determine if the action should be triggered or not and this article will mainly focus one those.  

A rule can be as simple as:

{% highlight xml %}
<rule name="Redirect to github">
  <match url="^code$" />
  <action type="Redirect" url="https://github.com" />
</rule>
{% endhighlight %}

This rule checks if the requested path is exactly `code` (as in the URL http://website.com/code) and if it is the case, redirects the user to <https://github.com>

## Back references

Now, the interesting part is when you start using back references. If you want your action to depend on a part (or all) of the requested URL, you can use back references to recall those values.

Let's look at the following example:

{% highlight xml %}
<rule name="Redirect to backreference">
  <match url="^(.*)$" />
  <action type="Redirect" url="https://{R:1}.com" />
</rule>
{% endhighlight %}

If we look at the pattern being used: `^(.*)$`:

  - The first character `^` means that the test will start at the beginning of the path.
  - The following sequence `(.*)` will capture any character.
  - The last character `$` means that the test stops at the end of the path.

Important: The rule is only applied to the path; don't let the name `url` fool you. (for example, in http://example.com/test, the scheme and domain name are ignored for the "url" matching)

We now want to use the value captured in the test for the redirect. That's when the back reference, `{R:1}`, comes to help.  
The `{R:1}` part will contain whatever is captured inside the `(.*)` sequence during the test.  
We could use `{R:0}` as well, since is contains the whole input string.

If we apply for example the URL http://website.com/github to this rule, `{R:1}` will contain `github`.  
It means that combined with the action, a user reaching http://website.com/github will ultimately be redirected to <https://github.com>

Taking back the example from my answer on [stackoverflow]:

Using the pattern `^(www\.)(.*)$` and the input string `www.foo.com`, you will have:

    {R:0} - www.foo.com
    {R:1} - www.
    {R:2} - foo.com

The `0` reference always contains the full input while the `1` will contain the first part of the string matching the pattern in the first parenthesis `()`, the 2 reference the second one, etc...up to the reference number `9`.

### Conditions

When you add conditions to your rule, you get access to more data from the request.  
Let's one more time take an example:

{% highlight xml %}
<rule name="Redirect to backreference with domain">
  <match url="^(.*)$" />
  <conditions>
      <add input="{HTTP_HOST}" pattern="^.+$" />
  </conditions>
  <action type="Redirect" url="https://{R:1}.com?ref={C:0}" />
</rule>
{% endhighlight %}

This rule does the same as the previous one but it affects the original requested domain name to a `ref` variable in the querystring.
We get the domain name by using the pattern `^.+$` (one or more characters) against the `{HTTP_HOST}` input.
We are using `{C:0}` in this case. `C` stands for `Conditions` (against `R` for `Rule`) and `0` simply means the whole input value.

You can find the list of server variables and their documentation here:  
<https://msdn.microsoft.com/en-us/library/ms524602(v=vs.90).aspx>  
(it says IIS 6 but it applys to higher versions as well)

#### Note:

  - you can use the captured value in the rule (for example `{R:1}`) inside conditions.
  - if you have multiple conditions, the back reference used in the action refers only to the last **matching** one.
  - you can however use a back reference from a previous condition inside one.

{% highlight xml %}
<rule name="Redirect to backreference with domain">
  <match url=".*" />
  <conditions>
      <add input="{HTTP_HOST}" pattern="^.+$" />
      <add input="{C:0}_{QUERY_STRING}" pattern="^.+$" />
  </conditions>
  <action type="Redirect" url="https://example.com?ref={C:0}" />
</rule>
{% endhighlight %}

In this case, the back reference `{C:0}` used in the action contains the last matched condition (`<add input="{C:0}_{QUERY_STRING}" pattern="^.+$" />`) which itself contains the value of the previous condition and the requested querystring.

## How to debug

Debugging the rewrite rules can be tricky and at time annoying...but there's a very good tool available with the module called the [Failed Request Tracing] tool.  
Always remember when you debug a redirect (specifically a 301) that browsers tend to cache them and that it can lead to frustration when you change the rule but nothing happens... 

## Rewrite outside websites

There's one usage that's not very well documented and that you might face one day: rewriting to an external website.  
If you want http://website.com/code to rewrite to <https://github.com>, you can use the following rule:

{% highlight xml %}
<rule name="Rewrite to github">
  <match url="^code$" />
  <action type="Rewrite" url="https://github.com" />
</rule>
{% endhighlight %}

But if you try it, it will most likely not work as expected (read: it won't show he content of <https://github.com>).
It is because you need to install the [Application Request Routing module] and set the proxy mode to `enable`.

*There is much more to this module and I strongly encourage you to read the official documentation: <http://www.iis.net/learn/extensions/url-rewrite-module/url-rewrite-module-configuration-reference>*

[stackoverflow]: http://stackoverflow.com/a/17010848/1443490
[Application Request Routing module]: http://www.iis.net/downloads/microsoft/application-request-routing
[Failed Request Tracing]: http://www.iis.net/learn/extensions/url-rewrite-module/using-failed-request-tracing-to-trace-rewrite-rules