---
layout: post
title:  "The C# parameterless default constructor - second edition"
date:   2018-04-25 23:00:00
categories: blog
tags: C# constructor parameterless reflection default value generics rant
---

[Back in 2015](/blog/2015/03/29/c-sharp-constructor.html) I wrote an article about the C# default parameterless constructor and how it goes away if you write one with default values for the parameters.

I have since seen this behavior more and more and I actually spent more time understanding what was going on.

It turns out that my assumption was wrong ([see the older article](/blog/2015/03/29/c-sharp-constructor.html)) and the explanation is actually simpler:

When a default value is set to a parameter, it is actually replaced at compile time where the calls don't set the parameters.

So now the IL looks like you wrote the code by setting the default values where you call the empty construcors/methods.

We will take the following example:

{% highlight C# %}
class Foo
{
    private readonly int _bar;

    public Foo(int bar = 22)
    {
        _bar = bar;
    }
}
{% endhighlight %}

Fairly straight forward example. Let's look at the IL for the constructor:

{% highlight C# %}
.method public hidebysig specialname rtspecialname 
        instance void  .ctor([opt] int32 bar) cil managed
{
  .param [1] = int32(0x00000016)
  // Code size       16 (0x10)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  call       instance void [System.Runtime]System.Object::.ctor()
  IL_0006:  nop
  IL_0007:  nop
  IL_0008:  ldarg.0
  IL_0009:  ldarg.1
  IL_000a:  stfld      int32 sandbox_console.Foo::_bar
  IL_000f:  ret
} // end of method Foo::.ctor
{% endhighlight %}

What's going on here? Well, the CLR is given a hint that the parameter `bar` is optionnal with `[opt]`.
And the default value is given on the following line: `.param [1] = int32(0x00000016)` (0x00000016 is 22 in base 10)

As we can see, the compiler does not create a new parameterless constructor calling the other one with the default value.

The reason was explained by Eric Lippert over here (the 4 posts are definitely worth a read): <https://blogs.msdn.microsoft.com/ericlippert/2011/05/16/optional-argument-corner-cases-part-three/>

I already mentionned how it was kind of annoying with AutoMapper ([in the previous article](/blog/2015/03/29/c-sharp-constructor.html)) and I actually have a case that I run into more often that is equally annoying.

Let's say you have a class like this:

{% highlight C# %}
class ItemFactory<T> where T : new()
{
    public T GetNewItem()
    {
        return new T();
    }
}
{% endhighlight %}

The IL for the GetNewItem method will look like this:

{% highlight C# %}
.method public hidebysig instance !T  GetNewItem() cil managed
{
  // Code size       11 (0xb)
  .maxstack  1
  .locals init (!T V_0)
  IL_0000:  nop
  IL_0001:  call       !!0 [System.Runtime]System.Activator::CreateInstance<!T>()
  IL_0006:  stloc.0
  IL_0007:  br.s       IL_0009
  IL_0009:  ldloc.0
  IL_000a:  ret
} // end of method ItemFactory`1::GetNewItem
{% endhighlight %}

As you can see, the compiler is also using the `Activator.CreateInstance<T>()` (because `new T()` wouldn't be correct) in this case so the parameterless constructor is required.
(it wouldn't let you compile if it wasn't the case anyway since the constraint `new()` forces you to have a parameterless public constructor).

I'll write the next few articles about features I wish C# was handling differently.
