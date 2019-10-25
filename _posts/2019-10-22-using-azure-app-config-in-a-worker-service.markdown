---
layout: post
title: "Using Azure App Configuration in a Worker Service"
date: 2019-10-22
tags: c# azure .net-core worker-service app-configuration
---

<p class="intro"><span class="dropcap">W</span>ith .NET Core 3 we were introduced to the new Worker Service template for running continuous jobs. In this article I'll show you how to setup the service and also to make use of the new Azure App Configuration service that centralizes the management of app settings.</p>

<p>The Worker Service is simple to use and easy to understand - just create an app with the template and run it. However, it lacks some of the features the classing web job had (timers, queue handling etc) and instead you have to wire up this functionality yourself. When you create a new Worker Service template you're presented with a list page currently only having one option - the basic temple, so we can probably expect more Worker Service templates coming here in the future. Time will tell if that assumption is correct or not.</p>

<p>The Azure App Configuration service enables admins to centralize app settings for several services over multiple environments. I'll write another article about specific use cases here later on.</p>

### Configure Azure App Configuration

<p>One of the best features in Azure App Configuration is labels. You can create a setting and then use labels to keep different values for the same setting separated. In the example below, I have created two settings configured for two different services, each with its own values, by the use of labels.</p>

{%
  include image.html
  url="2019-11-05-worker-service-app-config/2019_app-configuration-settings.png"
  alt=""
  description=""
%}

The keys use the standard way of using <code class="code">:</code> to create hierarchical values. You can read more about that in this article about [configuration providers]({{ site.baseurl }}{% post_url 2019-04-02-configuration-providers-in-aspnet-core %}).

### Creating a new Worker Service project

<p>To use the new Worker Service template you need to have <code class="code">.NET Core 3 SDK</code> installed on your computer. If you use Visual Studio then you need to use version 2019 (version 16.3 or higher), which also installs the correct .NET Core framework.</p>

<p>After you've created a new project using the Worker Service template, you also need to install the NuGet package <code class="code">Microsoft.Extensions.Configuration.AzureAppConfiguration</code> to be able to communicate with the App Config service.</p>

<p>Add the following settings to your <code class="code">appsettings.json</code> file. The connection string can be found under Access Keys in the App Configuration service. The hierarchical path to the settings OptionA and OptionB should match what you setup in App Configuration.</p>

{% highlight json linenos %}
{
  "AppConfiguration": {
    "ConnectionString": "..."
  },
  "Processing": {
    "OptionA": "blue",
    "OptionB": "green"
  },
  ...
}
{% endhighlight %}

<p>We also create a settings class that maps to these settings in appsettings.json</p>

{% highlight csharp linenos %}
public class ProcessingSettings
{
  public string OptionA { get; set; }
  public string OptionB { get; set; }
}
{% endhighlight %}

<p>In the <code class="code">Program.cs</code> file, extend the host builder with a call to <code class="code">ConfigureAppConfiguration</code> according to the example below.</p>

{% highlight csharp linenos %}
public class Program
{
  public static void Main(string[] args)
  {
    CreateHostBuilder(args).Build().Run();
  }

  public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
      .ConfigureAppConfiguration((hostingContext, config) =>
      {
        var settings = config.Build();
        config.AddAzureAppConfiguration(options =>
        {
          options.Connect(settings["AppConfiguration:ConnectionString"])
            .ConfigureRefresh(refresh =>
            {
              refresh.Register("Processing:OptionA", "InvoiceService")
                     .Register("Processing:OptionB", "InvoiceService");
            });
        });
      });
}
{% endhighlight %}

<p>At this point we've configured our app to first inject settings from appsettings.json and then to download the settings from Azure App Configuration every time we start the app. The order in which we specify the configuration providers here make a difference since a latter one will overwrite previously injected settings.</p> 

<p>If we then create a simple background service we can inject the settings by using the <code class="code">IOption</code> pattern.</p>

{% highlight csharp linenos %}
namespace InvoiceService
{
  public class InvoiceWorker : BackgroundService
  {
    private ProcessingSettings _settings;

    public InvoiceWorker(IOptions<ProcessingSettings> settings)
    {
      _settings = settings.Value;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
      while (!stoppingToken.IsCancellationRequested)
      {
        Console.WriteLine($"Option A: {_settings.OptionA}");
        Console.WriteLine($"Option B: {_settings.OptionB}");
        await Task.Delay(5000, stoppingToken);
      }
    }
  }
}
{% endhighlight %}

<p>This works great! But there's more we can do here to extend the experience. In an ASP.NET MVC Core application, we can use <code class="code">IOptionSnapshot</code> to fetch the settings anew each time the web app is being called. The key here is that <code class="code">IOptionSnapshot</code> checks for new values from the source each time it's being resolved. To avoid having each web app call generate a call to App Configuration, the client caches the settings for 30 sec by default.</p>

<p>However, in the worker service we only call the constructor once - when the app is starting, so that logical flow wouldn't work here. Instead we have to setup a loop that calls a refresh function at given intervals. We also need to change to use the the <code class="code">IOptionMonitor</code> which has a neat callback function that will capture all changes to the settings.</p>

<p>Change the <code class="code">Program.cs</code> file to look like the example below. The key in this change is on line 25 where we fetch a refresher object from the app configuration client that we later on can use to refresh data from the Azure service. Further down, we have a timer that forces the settings to refresh every 30 seconds. Notice how we also set the cache expiration to be a lower value than the default 30 seconds since this caching time has precedence over the refresh calls.</p>

{% highlight csharp linenos %}
public class Program
{
  private static IConfigurationRefresher _refresher = null;
  private static Timer _timer;

  public static void Main(string[] args)
  {
    CreateHostBuilder(args).Build().Run();
  }

  public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((hostingContext, config) =>
    {
      var settings = config.Build();
      config.AddAzureAppConfiguration(options =>
      {
          options.Connect(settings["AppConfig:ConnectionString"])
            .ConfigureRefresh(refresh =>
            {
                refresh.Register("Processing:OptionA", "InvoiceService")
                        .Register("Processing:OptionB", "InvoiceService")
                        .SetCacheExpiration(TimeSpan.FromSeconds(10));
            });
          _refresher = options.GetRefresher();

          _timer = new Timer(async (o) =>
          {
            await Program._refresher.Refresh();
          }, null, TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(30));
        });
      });
  }
}
{% endhighlight %}

<p>In our background worker we have to change <code class="code">IOption</code> to <code class="code">IOptionMonitor</code> to be able to detect the changes when a setting is changed in App Configuration.</p>

{% highlight csharp linenos %}
public CustomerWorker(IOptionsMonitor<ProcessingSettings> settings)
{
  settings.OnChange((settings) => {
    _settings = settings;
  });
  _settings = settings.CurrentValue;
}
{% endhighlight %}

<p>The <code class="code">OnChange</code> method will here be triggered every time the <code class="code">Refresh()</code> function will detect new values in App Configuration</p>

### Conclusions

<p>That's all we need to do. We're all hooked up to the App Configuration service and ever time the settings are being changed there, our Worker Service will pick it up within 30 seconds without needing a restart. This is a very neat functionality that you probably should implement whenever possible.</p>