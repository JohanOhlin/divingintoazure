---
layout: post
title: "Worker Service "
date: 2019-10-05
tags: c# azure .net-core worker-service
---

<p class="intro"><span class="dropcap">T</span>he Worker Service in .NET Core 3... In this article we'll go through how to create a worker service and use it in different scenarios</p>

### Create the app and configure logging

<p>Create a new app and add NuGet packages needed</p>

{% highlight bash linenos %}
dotnet new worker -o MyWorker
{% endhighlight %}

<p>Open the project file and add a reference to Application Insights for Worker Service.</p>

{% highlight xml linenos %}
  <ItemGroup>
    <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.8.2" />
  </ItemGroup>
{% endhighlight %}

<p>Install new NuGet packages</p>

{% highlight bash linenos %}
  dotnet restore
{% endhighlight %}

<p>Add the intrumentation key to the <code class="code">appsettings.json</code> file. If you don't have one then you need to create a new Application Insight service in Azure.</p>

{% highlight json linenos %}
  "ApplicationInsights": {
    "InstrumentationKey": "instrumentation key goes here"
  }
{% endhighlight %}

<p>The instrumentation key can be replaced in the deployment process by setting <code class="code">APPINSIGHTS_INSTRUMENTATIONKEY</code> or <code class="code">ApplicationInsights:InstrumentationKey</code> as environment variables</p>

<p>Add <code class="code">AddApplicationInsightsTelemetryWorkerService</code> to the <code class="code">ConfigureServices</code> section as shows in this example</p>

{% highlight bash linenos %}
public static IHostBuilder CreateHostBuilder(string[] args) =>
  Host.CreateDefaultBuilder(args)
      .ConfigureServices((hostContext, services) =>
      {
          services.AddHostedService<Worker>();
          services.AddApplicationInsightsTelemetryWorkerService();
      });
{% endhighlight %}

<p>Logging is now completely setup in your worker service. To use it you have two options:</p>

<ul>
  <li><b>Standard .NET Core logging using the ILogger framework</b> - all the logged events with severity above the configured threshold will be forwarded to application insights.</li>
  <li><b>TelemetryClient</b> - this native Application Insight client gives you more control over the logging if you want to use non-standard log events. This is also useful when you want to write a successful event to the log while the standard logging is set to only report on warnings and higher.</li>
</ul>

<p>The TelemetryClient and ILogger are setup as per below</p>

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

<p>You can read more about using logging together with Azure Monitor in this article</p>

### Processing a queue


### Timer processing

