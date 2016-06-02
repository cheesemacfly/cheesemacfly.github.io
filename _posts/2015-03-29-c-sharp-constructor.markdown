---
layout: post
title:  "The C# parameterless default constructor"
date:   2015-03-29 22:00:00
categories: blog
tags: C# constructor parameterless reflection default automapper
---

There's not many things in C# that I wish were different, but this is one of them.
If you define a `class`, it comes with a default parameterless constructor if you don't define one:

> If a class contains no instance constructor declarations, a default instance constructor is automatically provided.
> That default constructor simply invokes the parameterless constructor of the direct base class.

<https://msdn.microsoft.com/en-us/library/aa645608%28v=vs.71%29.aspx>

So this class:

{% highlight C# %}
public class Foo
{
    private string Bar;
}
{% endhighlight %}

is equivalent to this one:

{% highlight C# %}
public class Foo
{
    private string Bar;
    public Foo() : base() { }
}
{% endhighlight %}

So far so good. Now if you have a constructor with a parameter taking a default value, this rule doesn't apply anymore.

In the following example, the parameterless contructor is not automatically provided:

{% highlight C# %}
public class Foo
{
    private string Bar;
    public Foo(string bar = null)
    {
        Bar = bar;
    }
}
{% endhighlight %}

In most cases, you would be creating an instance using `new` as in `var foo = new Foo()` and shouldn't care much about the missing default constructor.
But if you are using reflection as in `var o = Activator.CreateInstance(typeof(Foo));`, you will be out of luck. Doing so will give you a `System.MissingMethodException, No parameterless constructor defined for this object.`

This is a problem I often run into when creating a custom resolver for [AutoMapper] (implementing the [`IValueResolver`] interface) for example.

Your only option is to recreate the parameterless constructor:

{% highlight C# %}
public class Foo
{
    private readonly string Bar;
    public Foo(string bar = null)
    {
        Bar = bar;
    }
    public Foo() : this(null) { } //default constructor
}
{% endhighlight %}

The reasoning behind this is probably that the default parameterless constructor calls the base class one and for consistency it shouldn't call anything else (because this is how it was defined).
Still would be a nice to have...


[AutoMapper]: http://automapper.org/
[`IValueResolver`]: https://github.com/AutoMapper/AutoMapper/wiki/Custom-value-resolvers