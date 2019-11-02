---
layout: post
title: "Worker Service "
date: 2019-10-05
tags: c# azure .net-core worker-service
---

<p class="intro"><span class="dropcap">T</span>he Worker Service in .NET Core 3... In this article we'll go through how to create a worker service and use it in different scenarios</p>

### Create the app and configure logging

Create a new app and add NuGet packages needed

{% highlight bash linenos %}
dotnet new worker -o MyWorker
{% endhighlight %}

Open the project file and add a reference to Application Insights for Worker Service.

{% highlight xml linenos %}
  <ItemGroup>
    <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.8.2" />
  </ItemGroup>
{% endhighlight %}

Install new NuGet packages

{% highlight bash linenos %}
  dotnet restore
{% endhighlight %}

Add the intrumentation key to the `appsettings.json` file. If you don't have one then you need to create a new Application Insight service in Azure.

{% highlight json linenos %}
  "ApplicationInsights": {
    "InstrumentationKey": "instrumentation key goes here"
  }
{% endhighlight %}

The instrumentation key can be replaced in the deployment process by setting `APPINSIGHTS_INSTRUMENTATIONKEY` or `ApplicationInsights:InstrumentationKey` as environment variables

Add `AddApplicationInsightsTelemetryWorkerService` to the `ConfigureServices` section as shows in this example

{% highlight bash linenos %}
public static IHostBuilder CreateHostBuilder(string[] args) =>
  Host.CreateDefaultBuilder(args)
      .ConfigureServices((hostContext, services) =>
      {
          services.AddHostedService<Worker>();
          services.AddApplicationInsightsTelemetryWorkerService();
      });
{% endhighlight %}

Logging is now completely setup in your worker service. To use it you have two options:

<ul>
  <li><b>Standard .NET Core logging using the ILogger framework</b> - all the logged events with severity above the configured threshold will be forwarded to application insights.</li>
  <li><b>TelemetryClient</b> - this native Application Insight client gives you more control over the logging if you want to use non-standard log events. This is also useful when you want to write a successful event to the log while the standard logging is set to only report on warnings and higher.</li>
</ul>

The TelemetryClient and ILogger are setup as per below

{% highlight csharp linenos %}
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private TelemetryClient _telemetryClient;

    public Worker(ILogger<Worker> logger, TelemetryClient tc)
    {
        _logger = logger;
        _telemetryClient = tc;
    }
}
{% endhighlight %}

You can read more about using logging together with Azure Monitor in this article

### Processing a queue


### Timer processing

