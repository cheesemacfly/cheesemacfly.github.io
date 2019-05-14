---
title: The C# parameterless default constructor
date: 2015-03-29 22:00:00 Z
categories:
- blog
tags:
- C#
- constructor
- parameterless
- reflection
- default
- automapper
layout: post
---

There's not many things in C# that I wish were different, but this is one of them.
If you define a `class`, it comes with a default parameterless constructor if you don't define one:

> Unless the class is static, classes without constructors are given a public default constructor by the C# compiler in order to enable class instantiation.

<https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/using-constructors>

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

<s>The reasoning behind this is probably that the default parameterless constructor calls the base class one and for consistency it shouldn't call anything else (because this is how it was defined).
Still would be a nice to have...</s>
<br/>

__Updated in April 2018:__  

I have posted a new post about why the last line was actually a wrong assumption. You can read it [here](/blog/2018/04/25/c-sharp-optionnal-arguments-second-edition.html).


[AutoMapper]: http://automapper.org/
[`IValueResolver`]: https://github.com/AutoMapper/AutoMapper/wiki/Custom-value-resolvers