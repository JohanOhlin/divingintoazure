---
layout: post
title: "Configuration in ASP.NET Core"
date: 2019-03-25
tags: asp.net-core .net-core
blog_serie: configuration_aspnet_core
---

<p class="intro"><span class="dropcap">C</span>onfiguration in your app can sometimes be a pain. Some values are set in code, some are extracted from the environment in which you're running and some are secret and can't be defined anywhere near a repository. ASP.NET Core helps you manage all these settings in a very smooth way</p>

{%
  include blog_serie.html
  page=page
%}

ASP.NET Core has a pluggable configuration system where configuration settings can be imported from a number of [different sources]({{ site.baseurl }}{% post_url 2019-04-02-configuration-providers-in-aspnet-core %}). These sources are then merged together into one hierarchical configuration object.

When you create a new ASP.Net Core Web App in Visual Studio, it'll automatically create a configuration file for you named <code class="code">appsettings.json</code>. By default it only contains logging and CORS settings but we're free to add any type of hierarchical configuratin that we need.

In this example I've added two extra sections (<i>FirstServiceSettings</i> and <i>SecondServiceSettings</i>) to the file that I'll use as specific option groups for my web app.

{% highlight json linenos %}
{
  "FirstServiceSettings": {
    "AnotherSetting": 14
  },
  "SecondServiceSettings": {
    "MinRetries": 2,
    "MaxRetries": 40,
    "CostPerRetry": 0.02
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AllowedHosts": "*"
}
{% endhighlight %}

#### Options pattern

Settings are grouped together as classes in what's called the <i>options pattern</i>. By separating options you can decouple different parts of your app so it only knows the settings it needs to know, in accordance with the theory of [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns){:target="_blank" rel="noopener"}.

It's not mandatory to create options classes for your options but it's something you should consider. Here's an example of a few settings classes that we'll use in the examples through out this post.

{% highlight csharp linenos %}
public class FirstServiceSettings
{
  public string StringSetting { get; set; }
  public int AnotherSetting { get; set; }
}

public class SecondServiceSettings
{
  public int MaxRetries { get; set; }
  public int MinRetries { get; set; }
  public decimal CostPerRetry { get; set; }
}
{% endhighlight %}

Since the JSON file example previously shown only is one of many possible configuration providers, it's not needed to include all the settings you need for your option classes. The JSON file might specify some basic values while a secrets manager brings in passwords.

#### Map classes to configuration

In the merged configuration settings, each option group that match a class is seen as a sub-option in the main hierarchy. In <code class="code">Startup.cs</code> we therefore need to map a path to each of the option classes.

{% highlight csharp linenos %}
public Startup(IConfiguration configuration)
{
  Configuration = configuration;
}

public IConfiguration Configuration { get; }

public void ConfigureServices(IServiceCollection services)
{
  services.Configure<FirstServiceSettings>(Configuration.GetSection("FirstServiceSettings"));
  services.Configure<SecondServiceSettings>(Configuration.GetSection("SecondServiceSettings"));
  ...
}
{% endhighlight %}

The option classes are now registered and we can use them in the code by [injecting them using dependency injection]({{ site.baseurl }}{% post_url 2019-03-31-injecting-options-in-aspnet-core %}).

#### Make options read-only

One problem with these option classes is that they are not read-only and can be changed locally where used (but still not globally). It is not possible for the configuration binder to set properties without a setter, but you can define the setter as private and still set these properties.

In this option class example below, both the public property with a private setter as well as the private property will be set with the values from the configuration settings.

{% highlight csharp linenos %}
public class ProtectedSettings
{
  public string HiddenSetting { get; private set; }
  private string VeryHiddenSetting { get; set; }
}
{% endhighlight %}

The default behaviour is to not change private properties, but by specifying the <code class="code">BindNonPublicProperties</code> option you can override this behaviour.

{% highlight csharp linenos %}
public void ConfigureServices(IServiceCollection services)
{
  services.Configure<ProtectedSettings>(Configuration.GetSection("ProtectedSettings"), options =>
  {
      options.BindNonPublicProperties = true;
  });
  ...
}
{% endhighlight %}

The option values are now protected later on.

#### Configure options using a delegate

It's also possible to set the configuration options in code by using a delegate. 

{% highlight csharp linenos %}
services.Configure<SecondServiceSettings>(myOptions =>
{
    myOptions.MaxRetries = 55;
});
{% endhighlight %}

If you want to use both delegate and the configuration object when you set the values from the settings class you have to write a little extra code. The example below shows two ways of accessing specific values in the configuration object.

{% highlight csharp linenos %}
services.Configure<SecondServiceSettings>(delegateSettings => 
{
  delegateSettings.MaxRetries = 55;

  // Deep link to values one by one
  delegateSettings.MinRetries = Configuration.GetValue<int>("SecondServiceSettings:MinRetries");

  // Get the whole settings object and set them one by one
  var settings = new SecondServiceSettings();
  Configuration.Bind("SecondServiceSettings", settings);
  delegateSettings.CostPerRetry = settings.CostPerRetry;
});
{% endhighlight %}

You can of course also use a model mapper like AutoMapper, but we're here quite early in the startup process so you might not want to do that here.

The delegate is not executed until the value object is requested in the injected option.

{% highlight csharp linenos %}
public class HomeController : Controller
{
  private readonly SecondServiceSettings _secondSettings;

  public HomeController(IOptionsSnapshot<SecondServiceSettings> secondSettings)
  {
      _secondSettings = secondSettings.Value;
  }
  ...
}
{% endhighlight %}

In the example above, line 7 would trigger the delegate in the configuration.

#### More information

More detailed information about other useful scenarios can be found on the Microsoft Docs site under the topics [options](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.2){:target="_blank" rel="noopener"} and [configuration](https://docs.microsoft.com/en-gb/aspnet/core/fundamentals/configuration/index?view=aspnetcore-2.2){:target="_blank" rel="noopener"}.