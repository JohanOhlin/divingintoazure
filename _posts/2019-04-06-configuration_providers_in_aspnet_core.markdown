---
layout: post
title: "Configuration providers in ASP.NET Core"
date: 2019-04-01
tags: asp.net-core .net-core
blog_serie: configuration_aspnet_core
---

<p class="intro"><span class="dropcap">C</span>onfiguration providers in ASP.NET Core offers a pluggable architecture for using many different sources for your app settings.</p>

{%
  include blog_serie.html
  page=page
%}

This post builds upon two previous posts concerning [basic configuration]({{ site.baseurl }}{% post_url 2019-04-06-configuration-in-asp-net-core %}) and [injecting options]({{ site.baseurl }}{% post_url 2019-04-06-injecting-options-in-aspnet-core %}).

#### Configuration providers

The configuration can be read from several different providers. These providers all provide key-value pairs of data that will be merged together.

Provider | Provides configuration from | Applied by default
--- | --- | ---
Azure Key Vault Configuration Provider (Security topics) | Azure Key Vault | Yes
Command-line Configuration Provider | Command-line parameters | Yes
Custom configuration provider | Custom source | No
[Environment Variables Configuration Provider](#environment-variables) | Environment variables | Yes (prefixed only)
[File Configuration Provider](#file-configuration) | Files (INI, JSON, XML) | Yes
Key-per-file Configuration Provider | Directory files | No
Memory Configuration Provider | In-memory collections | No
User secrets (Secret Manager) (Security topics) | File in the user profile directory | Yes

Not all of these are applied by default. Some has to be manually configured and enabled. You can read more about how that's done [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2#environment-variables-configuration-provider){:target="_blank" rel="noopener"}.

When more than one configuration service is enabled, the last configuration source specified wins and sets the configuration value.

#### File Configuration

Read this blog post for examples of how to use this 

ASP.NET Core also searches for a settings file with the current environment in the name, as in <code class="code">appsettings.{Environment}.json</code>. The environment specific configuration file is imported after the generic one and will thus override the generic values.

By default, you have <code class="code">Development</code>, <code class="code">Staging</code> and <code class="code">Production</code> to choose from, but you can specify environments yourself if needed. These environments are written with the first character as upper case and the rest lower case. This is important if you are using the code in case sensitive environments, i.e. Linux.

#### Environment Variables

Per default, only environment variables prefixed with <code class="code">ASPNETCORE_</code> will be included. The prefix will be removed before the variables are used in the hierarchy. 

The prefix can be changed to a custom string if needed (see example [here](#customization)).

If you're using hierarchies in your configuration (as we used in [this]({{ site.baseurl }}{% post_url 2019-04-06-configuration-in-asp-net-core %}) example) then your keys need to be hierarchical also. To match the content of this appsettings.json file

{% highlight json linenos %}
{
  "FirstServiceSettings": {
    "AnotherSetting": 14,
    "StringSetting": "maybe"
  }
}
{% endhighlight %}

You need to write the environment variables this way

{% highlight bash linenos %}
ASPNETCORE_FirstServiceSettings__AnotherSetting=25
{% endhighlight %}

If you use Azure and specify app settings for your web app then these will be injected (and have the priority) as environment variables. When you click Save, the Azure App will be restarted and after that the changed environment variables will be available for the app. 

#### Customization

If you need to customize the orders in which the providers are processed (since an imported value will override any previously imported values) then you can change this in the <code class="code">Program.cs</code> file. The following example shows how you can override and reconfigure the existing configuration.

{% highlight csharp linenos %}
public class Program
{
  public static void Main(string[] args)
  {
      CreateWebHostBuilder(args).Build().Run();
  }

  public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
      .ConfigureAppConfiguration((hostingContext, config) =>
      {
        var env = hostingContext.HostingEnvironment;

        config.SetBasePath(Directory.GetCurrentDirectory());
        config.AddXmlFile("settings.xml", optional: false, reloadOnChange: false);
        config.AddJsonFile("application.json", optional: false, reloadOnChange: true);
        config.AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);
        config.AddEnvironmentVariables("ASPNETCORE_");
        config.AddCommandLine(args);
      })
      .UseStartup<Startup>();
}
{% endhighlight %}

It's important to set the base path as the first step so the files will be searched for in the right place.