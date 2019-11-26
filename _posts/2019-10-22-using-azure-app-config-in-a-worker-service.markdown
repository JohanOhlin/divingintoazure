---
layout: post
title: "Using Azure App Configuration in a Worker Service"
date: 2019-10-22
tags: c# azure .net-core worker-service app-configuration
blog_serie: configuration_aspnet_core
twitter-title: "Using #Azure app configuration in a #netcore Worker Service"
description: "With .NET Core 3 we were introduced to the new Worker Service template for running continuous jobs. In this article, I'll show you how to setup the service and also to make use of the new Azure App Configuration service that centralizes the management of app settings."
image: 2019-11-05-worker-service-app-config/levers.jpg
---

<p class="intro"><span class="dropcap">W</span>ith .NET Core 3 we were introduced to the new Worker Service template for running continuous jobs. In this article, I'll show you how to setup the service and also to make use of the new Azure App Configuration service that centralizes the management of app settings.</p>

{%
  include blog_serie.html
  page=page
%}

The Worker Service is simple to use and easy to understand - just create an app with the template and run it. However, it lacks some of the features the classing web job had (timers, queue handling etc) and instead you must wire up this functionality yourself. When you create a new Worker Service template you're presented with a list page currently only having one option - the basic temple, so we can probably expect more Worker Service templates coming here in the future. Time will tell if that assumption is correct or not.

The Azure App Configuration service enables admins to centralize app settings for several services over multiple environments. I'll write another article about specific use cases here later on.

### Configure Azure App Configuration

One of the best features in Azure App Configuration is labels. You can create a setting and then use labels to keep different values for the same setting separated. In the example below, I'm just creating two simple settings configured for two different services, each with its own values, using labels. In the code further down I'll connect and fetch the settings for one of these services.

{%
  include image.html
  url="2019-11-05-worker-service-app-config/2019_app-configuration-settings.png"
  alt=""
  description=""
%}

The keys use the standard way of using `:` to create hierarchical values. You can read more about that in this article about [configuration providers]({{ site.baseurl }}{% post_url 2019-04-02-configuration-providers-in-aspnet-core %}).

### Creating a new Worker Service project

To use the new Worker Service template, you need to have `.NET Core 3 SDK` installed on your computer. If you use Visual Studio then you need to use version 2019 (version 16.3 or higher), which also installs the correct .NET Core framework.

After you've created a new project using the Worker Service template, you also need to install the NuGet package `Microsoft.Extensions.Configuration.AzureAppConfiguration` to be able to communicate with the App Config service.

Add the following settings to your `appsettings.json` file. The connection string can be found under Access Keys in the App Configuration service. The hierarchical path to the settings OptionA and OptionB should match what you setup in App Configuration.

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

We also create a settings class that maps to these settings in appsettings.json

{% highlight csharp linenos %}
public class ProcessingSettings
{
  public string OptionA { get; set; }
  public string OptionB { get; set; }
}
{% endhighlight %}

In the `Program.cs` file, extend the host builder with a call to `ConfigureAppConfiguration` according to the example below.

{% highlight csharp linenos %}
public class Program
{
  public static void Main(string[] args)
  {
    CreateHostBuilder(args).Build().Run();
  }

  public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
      .ConfigureServices((hostContext, services) =>
      {
        // Default backup settings from appsettings.json
        var settings = hostContext.Configuration.GetSection("Processing");
        services.Configure<ProcessingSettings>(settings);

        // Register the worker service
        services.AddHostedService<CustomerWorker>();
      })
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

The `ConfigureRefresh` function deals with what will happen when a refresh of settings is triggered, something that will happen automatically at startup.

At this point we've configured our app to first inject settings from appsettings.json and then to download the settings from Azure App Configuration every time we start the app. The order in which we specify the configuration providers here make a difference since a latter one will overwrite previously injected settings. If all your settings come from Azure App Configuration, then you can exclude the line where you configure `ProcessingSettings` in the example above.

In this example, we're downloading the values for OptionA and OptionB for label InvoiceService. If you create a setting without a label, then just leave this variable out or pass in `\0` as label.

If we then create a simple background service, we can inject the settings by using the `IOption` pattern.

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

This works great! But there's more we can do here to extend the experience. In an ASP.NET MVC Core application, we can use `IOptionSnapshot` to fetch the settings anew each time the web app is being called. The key here is that `IOptionSnapshot` checks for new values from the source each time it's being resolved. To avoid having each web app call generate a call to App Configuration, the client caches the settings for 30 sec by default.

However, in the worker service we only call the constructor once - when the app is starting, so that logical flow wouldn't work here. Instead we have to setup a loop that calls a refresh function at given intervals. We also need to change to use the `IOptionMonitor` which has a neat callback function that will capture all changes to the settings.

Change the `Program.cs` file to look like the example below. The key in this change is on line 34 where we fetch a refresher object from the app configuration client that we later on can use to refresh data from the Azure service. Further down, we have a timer that forces the settings to refresh every 30 seconds. 

Notice how we also set the cache expiration to be a lower value than the default 30 seconds. This value basically sets how often resolving `IOptionSnapshot` should trigger a refresh from Azure App Configuration. For a worker service, having a high cache time here doesn't make sense since we're not using `IOptionSnapshot` to trigger a refresh but rather use a controlled timer. But we shouldn't neither keep it to the default 30 seconds because the cache still prevents us from seeing the refreshed values, thus we set it to a low value.

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
    .ConfigureServices((hostContext, services) =>
    {
      // Default backup settings from appsettings.json
      var settings = hostContext.Configuration.GetSection("Processing");
      services.Configure<ProcessingSettings>(settings);

      // Register the worker service
      services.AddHostedService<CustomerWorker>();
    })
    .ConfigureAppConfiguration((hostingContext, config) =>
    {
      var settings = config.Build();
      config.AddAzureAppConfiguration(options =>
      {
          options.Connect(settings["AppConfiguration:ConnectionString"])
            .ConfigureRefresh(refresh =>
            {
                refresh.Register("Processing:OptionA", "InvoiceService")
                        .Register("Processing:OptionB", "InvoiceService")
                        .SetCacheExpiration(TimeSpan.FromSeconds(1));
            });
          _refresher = options.GetRefresher();

          _timer = new Timer(async (o) =>
          {
            await Program._refresher.Refresh();
          }, null, TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
        });
      });
  }
}
{% endhighlight %}

In our background worker we must change `IOption` to `IOptionMonitor` to be able to detect the changes when a setting is changed in App Configuration.

{% highlight csharp linenos %}
public CustomerWorker(IOptionsMonitor<ProcessingSettings> settings)
{
  settings.OnChange((settings) => {
    _settings = settings;
  });
  _settings = settings.CurrentValue;
}
{% endhighlight %}

The `OnChange` method will here be triggered every time the `Refresh()` function will detect new values in App Configuration

### Conclusions

That's all we need to do. We're all hooked up to the App Configuration service and every time the settings are being changed there, our Worker Service will pick it up within 30 seconds without needing a restart. This is a very neat functionality that you probably should implement whenever possible.