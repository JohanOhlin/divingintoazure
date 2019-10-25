---
layout: post
title: "Re-throwing exceptions - the right way"
date: 2019-08-02
tags: c#
---

<p class="intro"><span class="dropcap">I</span>n the company where I work we have an interview question for candidates where we ask them to explain the difference between different ways to re-throw an exception in a C# try-catch expression. How hard can it be? Well, surprisingly many developers get it wrong.</p>

<p>The questions looks something like this:</p>

<p><i>In the code for <code class="code">MyBrokenFunction</code> below, three different ways to re-throw an exception are shown. Explain the difference between the three different options, what we'll see in the catch section in the <code class="code">Main</code> function, and which one is the best to use.</i></p>

{% highlight csharp linenos %}
using System;
using System.Diagnostics;

namespace ThrowErrorExample
{
    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                MyImportantFunction();
            }
            catch (Exception e)
            {
                // Log error or something
                Debugger.Break();
            }
        }

        static void MyImportantFunction()
        {
            try
            {
                var value = Divide(10, 0);
            }
            catch (Exception ex)
            {
                // Log exception or something

                // Option 1
                throw ex;

                // Option 2
                throw;

                // Option 3
                throw new Exception("Failed to do something important", ex);
            }
        }

        static double Divide(int a, int b)
        {
            return a / b;
        }
    }
}
{% endhighlight %}

<p>Do you know which one you would choose? Well, let's go through the different options one by one.</p>

### Option 1

<p>In this example we re-throw the same exception object that is passed in into the catch clause.</p>

{% highlight csharp linenos %}
try
{
  var value = Divide(10, 0);
}
catch (Exception ex)
{
  // Log exception or something
  throw ex;
}
{% endhighlight %}

<p>It looks straight forward, but it has a major drawback. Re-throwing the same exception object resets the stack trace so we won't be able to see the original error.</p>

{% highlight csharp linenos %}
at ThrowErrorExample.Program.MyImportantFunction() in C:\Users\me\ThrowErrorExample\Program.cs:line 32
at ThrowErrorExample.Program.Main(String[] args) in C:\Users\me\ThrowErrorExample\Program.cs:line 12
{% endhighlight %}

<p>By looking at the StackTrace, it looks like the exception started on line 32 and we have no information about earlier problems. The error message is "Attempted to divide by zero" which is absolutely correct, but the message is somewhat confusing since we have no direct code that is making any divisions.</p>

<p>This example is quite easy, but if you have a complex hierarchy with many layers of code, then you might have lost valuable information about where the problem was caused.</p>

### Option 2

<p>In the second example we simply throw without specifying what we throw.</p>

{% highlight csharp linenos %}
try
{
  var value = Divide(10, 0);
}
catch (Exception ex)
{
  // Log exception or something
  throw;
}
{% endhighlight %}

<p>The result in the stack trace is different from the previous one and we see no references at all to line 32 where the throw statement is.</p>

{% highlight csharp linenos %}
at ThrowErrorExample.Program.Divide(Int32 a, Int32 b) in C:\Users\me\ThrowErrorExample\Program.cs:line 44
at ThrowErrorExample.Program.MyImportantFunction() in C:\Users\me\ThrowErrorExample\Program.cs:line 25
at ThrowErrorExample.Program.Main(String[] args) in C:\Users\me\ThrowErrorExample\Program.cs:line 12
{% endhighlight %}

<p>Instead we see the full stack trace back to the line where the original problem occurred. As in the previous option, the error message is "Attempted to divide by zero", and here it makes more sense if we also look at the stack trace.</p>

### Option 3

<p>The final option is that we wrap the exception in a new exception, maybe a domain specific version that contains more information about the problem.</p>

{% highlight csharp linenos %}
try
{
  var value = Divide(10, 0);
}
catch (Exception ex)
{
  // Log exception or something
  throw new Exception("Failed to do something important", ex);
}
{% endhighlight %}

<p>The stack trace in the caught exception will the look like</p>

{% highlight csharp linenos %}
at ThrowErrorExample.Program.MyImportantFunction() in C:\Users\me\ThrowErrorExample\Program.cs:line 38
at ThrowErrorExample.Program.Main(String[] args) in C:\Users\me\ThrowErrorExample\Program.cs:line 12
{% endhighlight %}

<p>We have no information here about the original exception, the same problem as we had for option 1, but this time we have an inner exception as well with its own stack trace.</p>

{% highlight csharp linenos %}
at ThrowErrorExample.Program.Divide(Int32 a, Int32 b) in C:\Users\me\ThrowErrorExample\Program.cs:line 44
at ThrowErrorExample.Program.MyImportantFunction() in C:\Users\me\ThrowErrorExample\Program.cs:line 25
{% endhighlight %}

<p>This inner exception points us all the way back to the original exception source.</p>

<p>The first exception had the message specified when we created the new exception, but the inner exception had the original message</p>

### Conclusions

<p>Looking at the examples above, it's quite clear that option 2 is the preferred choice and option 1 should be avoided. Option 2 is also the default expression when you use the existing code snippets in Visual Studio.</p>

<p>Most developers we interviewed picked option 1 as their primary choice and option 2 as their second. If it's something I've learned from these interviews then it's that guessing based upon logic won't always take you in the right direction.</p>