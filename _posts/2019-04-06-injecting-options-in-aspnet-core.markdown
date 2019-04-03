---
layout: post
title: "Injecting Options in ASP.NET Core"
date: 2019-04-01
tags: asp.net-core
blog_serie: configuration_aspnet_core
---
 
<p class="intro"><span class="dropcap">I</span>n ASP.NET Core there are some very interesting ways of handling and injecting options into classes. In this post I compare some of the different options you have at your disposal.</p>

{%
  include blog_serie.html
  page=page
%}

In a [previous post]({{ site.baseurl }}{% post_url 2019-04-06-configuration-in-asp-net-core %}) I wrote the basics about how to configure options in ASP.NET Core. If you're not familiar with this topic then I suggest you read through that blog first.

#### Comparison

There are a few different ways to handle option injection in your classes. They are all quite straight forward to use.

Interface | Automatic reloading | Works in a singleton service
--- | --- | ---
IOption | No | Yes
IOption Monitor | Yes (but you have to hook into change event) | Yes
IOption Snapshot | Yes (re-calculated for each call) | No

#### IOption

If your options change seldom and you're OK with restarting your app when you change your app then this should be your choice. Options are read at startup and there's no overhead for re-reading the file or watching for changes for every request.

{% highlight csharp linenos %}
public class MyService
{
  private readonly MyServiceSettings settings;

  public MyService(IOptions<MyServiceSettings> options)
  {
    settings = options.Value;
  }
}
{% endhighlight %}

#### IOptionMonitor

This option keeps an eye on the file and calls <code class="code">OnChange</code> when the file is changed. This makes it possible for you to make changes to your <code class="code">application.json</code> file without having to reload the app. 

{% highlight csharp linenos %}
public class MyService
{
  private MyServiceSettings settings;

  public MyService(IOptionsMonitor<MyServiceSettings> options)
  {
    settings = options.CurrentValue;
    options.OnChange(OnSettingsChange);
  }

  private void OnSettingsChange(MyServiceSettings settings)
  {
    this.settings = settings;
  }
}
{% endhighlight %}

#### IOptionSnapshot

The last option gives us the possibility to re-read the settings file for every time the dependency injection is executed. No need to handle change requests since you'll always have a new file.

This is a scoped service and can't be accessed from a singleton service. This might not be a problem to you, but it's good to keep in mind. <code class="code">IOptionMonitor</code> works just fine in a singleton.

{% highlight csharp linenos %}
public class MyService
{
  private readonly MyServiceSettings settings;

  public MyService(IOptionsSnapshot<MyServiceSettings> options)
  {
    settings = options.Value;
  }
}
{% endhighlight %}

#### IConfiguration

It's also possible to inject the whole configuration object into your class.

{% highlight csharp linenos %}
public class HomeController : Controller
{
  private readonly SecondServiceSettings _settings;

  public HomeController(IConfiguration configuration)
  {
    _settings = new SecondServiceSettings();
    configuration.Bind("SecondServiceSettings", _settings);
  }
}
{% endhighlight %}

This is not recommended since it'll break the separation of concerns that you should only have the options available that you need.