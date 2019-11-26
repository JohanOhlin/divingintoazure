---
layout: post
title: "Worker services in Kubernetes with health checks"
date: 2019-11-07
tags: c# azure .net-core worker-service kubernetes
twitter-title: "#NetCore worker service in #Kubernetes with health checks"
description: "One of the key features of Kubernetes is its ability to kick out bad pods and start up new ones. But to be able to do that, Kubernetes needs to communicate with the service to determine its health status."
image: 2019-11-07-worker-service-in-kubernetes/health-check.jpg
---

<p class="intro"><span class="dropcap">O</span>ne of the key features of Kubernetes is its ability to kick out bad pods and start up new ones. But to be able to do that, Kubernetes needs to communicate with the service to determine its health status.</p>

In this first article, we're creating a .NET Core 3 Worker Service with the ability to report it's status and determine if it's good or bad. In the [next article]({{ site.baseurl }}{% post_url 2019-11-19-deploying-worker-service-to-kubernetes %}), we continue with deploying this worker service to AKS, enabling Kubernetes to detect if app is in a bad state and then restart it automatically.

To make a practical example out of this, we'll set the service to always throw an unhandled exception after some time causing the worker loop to stop. This is a typical example of when we want Kubernets to remove the bad service and start up a new one instead.

### Setting up the worker service

Per default, the worker service is only thinking about itself, running an internal job, processing a queue or something other less interactive. This is very different from a web service which constantly listens for incoming requests. To implement health checks we need to add light-weight web service capabilities to the worker service so Kubernetes can send requests to ask for health updates. There's no Visual Studio template that includes both worker and web service and you can solve this in different ways. In this article, I've started by creating a web service and then I added the functionality for the worker service, but you could do it the other way around also.

Create a new empty project using the `ASP.NET Core Web Application` template. Make sure to use ASP.NET Core 3.0 as framework. If you've installed Visual Studio 2019 (16.3 or higher) then 3.0 will be the default framework when creating new projects.

If you create projects using the command prompt then it can be done like this.

{% highlight bash linenos %}
dotnet new web -n WorkerServiceWithHealthChecks
{% endhighlight %}

Add the NuGet package `Microsoft.Extensions.Hosting` to your project.

Create a worker service as shown below. This code will run for 10 minutes and then throw an unhandled exception which will cause the service to fail, but the app will still be running.

{% highlight csharp linenos %}
public class InvoiceWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var ticks = 0;
        while (!stoppingToken.IsCancellationRequested)
        {
            if (ticks++ > 10) throw new Exception("Ups...");

            Console.WriteLine("Processing an invoice");
            await Task.Delay(60 * 1000, stoppingToken);
        }
    }
}
{% endhighlight %}

Register the worker service in the `ConfigureServices` method

{% highlight csharp linenos %}
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHostedService<InvoiceWorker>();
    }
{% endhighlight %}

Your worker service is now ready to be used.

### Adding a health check endpoint

The logic for supporting health checks in an ASP.NET Core Web App is built in and you don't need any extra NuGet packages for it. But you do need to wire up the internal logic - how you determine if your app is healthy or not. 

To do that, we start by creating a static class that we can report the worker service progress to.

{% highlight csharp linenos %}
public static class InvoiceWorkerStatistics
{
    private static DateTime _lastProcessTime;
    private static readonly object _lockObject = new object();
    public static DateTime GetLastProcessTime()
    {
        lock (_lockObject) {
            return _lastProcessTime;
        }
    }
    public static void SetProcessTime()
    {
        lock (_lockObject) {
            _lastProcessTime = DateTime.UtcNow;
        }
    }
}
{% endhighlight %}

In the `InvoiceWorker` class we created earlier, add the following line to update the statistics class every time we're running a loop in the iteration.

{% highlight csharp linenos %}
InvoiceWorkerStatistics.SetProcessTime();
{% endhighlight %}

Every 3 seconds, the worker service will now report progress until it crashes out after 10 iterations.

Next we create a health check class that can interpret these statistics and figure out if our worker service is in a healty state or not. The function checks the last time an iteration was running. If it was more than 90 seconds ago then something must have gone wrong and we deem the service to be unhealthy, otherwise it's healthy. You also have the option to report a `Degraded` result if the service temporarily needs a break but soon will be available again. For our worker service scenario, since we don't receive web requests for processing, this is not needed.

{% highlight csharp linenos %}
public class InvoiceWorkerHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var lastProcess = InvoiceWorkerStatistics.GetLastProcessTime();
        var timeAgo = DateTime.UtcNow.Subtract(lastProcess);
        var data = new Dictionary<string, object> { 
            { "Last process", lastProcess },
            { "Time ago", timeAgo }
        } as IReadOnlyDictionary<string,object>;

        if (lastProcess > DateTime.UtcNow.AddSeconds(-90))
        {
            return Task.FromResult(
                HealthCheckResult.Healthy("Processing as much as we can", data));
        }

        return Task.FromResult(
            HealthCheckResult.Unhealthy("Processing is stuck somewhere", null, data));
    }
}
{% endhighlight %}

So how long should you wait until classifying your worker service as unhealthy? That completely depends on what you're processing. If you have few processes that each take 10 minutes to process then maybe you need to wait 30 minutes until you report an unhealthy status. We'll talk more about how Kubernetes handles these values in the next article, but if an unhealthy status is reported, then Kubernetes will restart your worker service, so be careful with how you set the values here.

Configure the health check service in the `ConfigureServices` method

{% highlight csharp linenos %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
            .AddCheck<InvoiceWorkerHealthCheck>("service_health_check");
    services.AddHostedService<InvoiceWorker>();
}
{% endhighlight %}

Expose an endpoint for the health checks by making changes to the `Configure` method as shown below. In this example, I've added a custom [response writer](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.0#customize-output) that creates a JSON document with all the available data from the health check. If you don't add this then the defined health check message will be returned instead.

{% highlight csharp linenos %}
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHealthChecks("/health", new HealthCheckOptions()
        {
            AllowCachingResponses = false, 
            ResponseWriter = WriteResponse
        });
    });
}

private static Task WriteResponse(HttpContext httpContext, HealthReport result)
{
    httpContext.Response.ContentType = "application/json";

    var json = new JObject(
        new JProperty("status", result.Status.ToString()),
        new JProperty("results", new JObject(result.Entries.Select(pair =>
            new JProperty(pair.Key, new JObject(
                new JProperty("status", pair.Value.Status.ToString()),
                new JProperty("description", pair.Value.Description),
                new JProperty("data", new JObject(pair.Value.Data.Select(
                    p => new JProperty(p.Key, p.Value))))))))));
    return httpContext.Response.WriteAsync(
        json.ToString(Formatting.Indented));
}
{% endhighlight %}

Your health checks are now completely setup. You can test it by running the project and navigate to the health endpoint. In the beginning, you'll get an HTTP 200 response looking like this.

{% highlight json linenos %}
{
  "status": "Healthy",
  "results": {
    "service_health_check": {
      "status": "Healthy",
      "description": "Processing as much as we can",
      "data": {
        "Last process": "2019-11-01T13:42:21.5588159Z",
        "Time ago": "00:00:01.4801542"
      }
    }
  }
}
{% endhighlight %}

But after a minute, when the exception has been thrown and the timeout has passed, the response will be an HTTP 503 (Service Unavailable) looking like this.

{% highlight json linenos %}
{
  "status": "Unhealthy",
  "results": {
    "service_health_check": {
      "status": "Unhealthy",
      "description": "Processing is stuck somewhere",
      "data": {
        "Last process": "2019-11-01T13:42:45.7795177Z",
        "Time ago": "00:00:30.6624213"
      }
    }
  }
}
{% endhighlight %}

### Conclusions

It's very neat to be able to get health data from a worker service, or from any service for that matter. Can you always use health checks created this way in a worker service? The simple answer is no. When you process inside the worker service you're blocking the thread and no calls can be made to the health endpoint. If you have a long-running process that never uses any async/await calls then you'll block for a long time and you might get false health results due to timeouts on the health endpoint. 

The key here is if you can use async/await once in a while to free up the thread to process other things, like web requests. This could look something like this.

{% highlight json linenos %}
public class InvoiceWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Heavy process loop
            for (int i = 0; i < 100; i++) {
                externalService.ProcessNonAsyncCall();
                await Task.Delay(1, stoppingToken);
            }
            await Task.Delay(60 * 1000, stoppingToken);
        }
    }
}
{% endhighlight %}

So, even if the external service used inside the iterator isn't async, we can make an async call inbetween the process calls and thus free up resources for the health checks. It all depends on the specific scenarios for your worker service.

### Next article

The worker service is now completely configured for health checks. In the [next article]({{ site.baseurl }}{% post_url 2019-11-19-deploying-worker-service-to-kubernetes %}) we'll continue with deploying the worker service to kubernetes and to configure the usage of the health checks to determine if the pod is functioning or not.
