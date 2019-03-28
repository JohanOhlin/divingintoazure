---
layout: post
title: "Configuration in ASP.NET Core"
date: 2019-03-18
tags: asp.net-core
---

<p class="intro"><span class="dropcap">I</span>n ASP.NET Core there are some very interesting ways of handling and injecting options into classes. In this post I compare the different options you have at your disposal.</p>

#### Initial setup

Before we go into the different injection options you have, we first need to set the sceen with a simple example of how options are setup in ASP.NET Core.

It's not needed to create a class representing your configuration options, but it surely makes things easier later on when using the options. You can have a lot of different settings classes that are injected wherever needed.

{% highlight csharp linenos %}
public class MyServiceSettings
{
  public bool DefaultValue { get; set; }
  public int AnotherSetting { get; set; }
  public string StringSetting { get; set; }
}
{% endhighlight %}

The configuration can be read from several different sources:

Provider |Provides configuration from…
--- | ---
Azure Key Vault Configuration Provider (Security topics) | Azure Key Vault
Command-line Configuration Provider | Command-line parameters
Custom configuration provider | Custom source
Environment Variables Configuration Provider | Environment variables
File Configuration Provider | Files (INI, JSON, XML)
Key-per-file Configuration Provider | Directory files
Memory Configuration Provider | In-memory collections
User secrets (Secret Manager) (Security topics) | File in the user profile directory

In this example we'll just use a json file which is the standard way of doing it. One neat thing is that configuration settings can be read from several different sources and merged together. You can read about the other providers [here](https://docs.microsoft.com/en-gb/aspnet/core/fundamentals/configuration/index?view=aspnetcore-2.2).

A config file named <code class="code">appsettings.json</code> will automatically be searched for and imported as a settings file. We create that file and let the JSON structure mimic the configuration class previously created.

{% highlight json linenos %}
{
  "MyServiceSettings": {
    "DefaultValue": true,
    "AnotherSetting": 14,
    "StringSetting": "maybe"
  }
}
{% endhighlight %}

The <code class="code">appsettings.json</code> can contain settings for many different purposes, not just the option class we're adding. In Startup.cs we therefore need to map a path in this file to the option class.

{% highlight csharp linenos %}
private IConfiguration _configuration { get; }
public Startup(IConfiguration configuration)
{
  _configuration = configuration;
}

public void ConfigureServices(IServiceCollection services)
{
  services.Configure<MyServiceSettings>(_configuration.GetSection("MyServiceSettings"));
  ...
}
{% endhighlight %}

The options are now configured and we can use them in the code.

#### Comparison

There are theee different ways to handle option injection in your classes. They are all quite straight forward to use. 

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

#### More information

More detailed information about other useful scenarios can be found on the [Microsoft Docs site](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.1).