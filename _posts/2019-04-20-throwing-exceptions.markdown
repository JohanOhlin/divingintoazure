---
layout: post
title: "Throwing exceptions"
date: 2019-04-20
tags: c#
---

<p class="intro"><span class="dropcap">H</span>ave you ever seen catch code that re-throws the existing exception - I know I have. So what's wrong with that? I'll show you here.</p>

There are basically three ways to re-throw exceptions:
- Use <code class="code">throw;</code> without specifying an exception
- Re-throw the cought exception
- Wrap the cought exception in a new exception

All these three ways have their own behaviour and they don't give the same end result.

#### Throwing the same exception

T

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        try
        {
            GetQuotient(10, 0);
        }
        catch (System.Exception ex)
        {
            throw ex;
        }
    }

    static double GetQuotient(int dividend, int divisor)
    {
        return dividend / divisor;
    }
}
{% endhighlight %}

{% highlight csharp linenos %}
Unhandled Exception: System.DivideByZeroException: Attempted to divide by zero.
   at throwing_exceptions.Program.Main(String[] args) in C:\Dev\test\throwing-exceptions\Program.cs:line 15
{% endhighlight %}

#### Throwing

If we instead of re-throwing the same exception just use throw then the result becomes quite different.

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        try
        {
            GetQuotient(10, 0);
        }
        catch (System.Exception)
        {
            throw;
        }
    }

    static double GetQuotient(int dividend, int divisor)
    {
        return dividend / divisor;
    }
}
{% endhighlight %}

In the result below you can see that there are no references to the line of the throw statement. Instead, the error message only talks about where the error occured and the function that called the failing function. This is what you want in an error message.

{% highlight csharp linenos %}
Unhandled Exception: System.DivideByZeroException: Attempted to divide by zero.
   at throwing_exceptions.Program.GetQuotient(Int32 dividend, Int32 divisor) in C:\Dev\test\throwing-exceptions\Program.cs:line 21
   at throwing_exceptions.Program.Main(String[] args) in C:\Dev\test\throwing-exceptions\Program.cs:line 11
{% endhighlight %}

#### Throwing a wrapped exception

There are however times when you do what to throw an exception - when you wrap the cought exception inside another more specific exception. 

{% highlight csharp linenos %}
class Program
{
    static void Main(string[] args)
    {
        try
        {
            GetQuotient(10, 0);
        }
        catch (System.Exception ex)
        {
            throw new MathFailedException("Failed to get quotient", ex);
        }
    }

    static double GetQuotient(int dividend, int divisor)
    {
        return dividend / divisor;
    }
}
{% endhighlight %}

In the exception thrown we now get to see both the inner exception with all the initial failing data, as well as information about the wrapped exception. 

{% highlight csharp linenos %}
Unhandled Exception: System.Exception: Failed to get quotient ---> System.DivideByZeroException: Attempted to divide by zero.
   at throwing_exceptions.Program.GetQuotient(Int32 dividend, Int32 divisor) in C:\Dev\test\throwing-exceptions\Program.cs:line 21
   at throwing_exceptions.Program.Main(String[] args) in C:\Dev\test\throwing-exceptions\Program.cs:line 11
   --- End of inner exception stack trace ---
   at throwing_exceptions.Program.Main(String[] args) in C:\Dev\test\throwing-exceptions\Program.cs:line 15
{% endhighlight %}

#### Conclusions

The most preferred option is to wrap the exception in a new more specific exception. Some 

Re-throwing the existing is most of the times just wrong. So are there any occasions when you could justify re-throwing the existing error? If simply removing the stack trace but keeping the rest is what you want, this might be a choise for you. But it's still preferred to throw a new more specific exception.