---
layout: post
title: "Injecting Options in ASP.NET Core"
date: 2019-03-31
tags: asp.net-core
blog_serie: configuration_aspnet_core
description: "In ASP.NET Core there are some very interesting ways of handling and injecting options into classes. In this post I compare some of the different options you have at your disposal."
---
 
<p class="intro"><span class="dropcap">I</span>n ASP.NET Core there are some very interesting ways of handling and injecting options into classes. In this post I compare some of the different options you have at your disposal.</p>

{%
  include blog_serie.html
  page=page
%}

In a [previous post]({{ site.baseurl }}{% post_url 2019-03-25-configuration-in-asp-net-core %}) I wrote the basics about how to configure options in ASP.NET Core. If you're not familiar with this topic then I suggest you read through that blog first.

#### Comparison

There are a few different ways to handle option injection in your classes. They are all quite straight forward to use.

Interface | Automatic reloading | Works in a singleton service
--- | --- | ---
IOption | No | Yes
IOption Monitor | Yes (but you have to hook into change event) | Yes
IOption Snapshot | Yes (re-calculated for each call) | No

#### IOption

If your options change seldom and you're OK with restarting your app (or application pool) when you change your app settings then this could be your choice. Options are read at startup and there's no overhead for re-reading the file or watching for changes for every request.

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

This option keeps an eye on the file and calls `OnChange` when the file is changed. This makes it possible for you to make changes to your `application.json` file without having to reload the app.

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

`IOptionsMonitor` has a caching layer behind the scenes that stores the values. You can interact with this caching layer to clear the cache, forcing an update, but also to inject new values if needed. This will then trigger the `OnChange` event to be triggered in code where used.

#### IOptionSnapshot

The last option gives us the possibility to re-read the settings file for every new request. No need to handle change requests since you'll always have a new file.

This is a scoped service and can't be accessed from a singleton service. This might not be a problem to you, but it's good to keep in mind. `IOptionMonitor` works just fine in a singleton.

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

#### Which one to choose

If you read [Microsoft's documentation](https://docs.microsoft.com/en-gb/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.2#options-interfaces){:target="_blank" rel="noopener"} you can see that most focus is being put on the `IOptionMonitor` alternative. This is probably due to the interaction possibilities you have with it. However it comes with the need to write a few extra lines of code to make use of the change requests.

`IOption` and `IOptionSnapshot` are also good options but they offer less functionality, which still might be good enough for your needs.