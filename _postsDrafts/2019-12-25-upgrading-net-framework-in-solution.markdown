---
layout: post
title: "Upgrading ASP.NET Framework version for a whole Solution"
date: 2019-12-25
tags: c# asp.net
---

<p class="intro"><span class="dropcap">L</span>et's say you have a legacy solution with 200 projects. Most of these use a not so new version of .NET Framework. Chances are, this will then prevent you from using modern security features, like TLS 1.2 etc. How do you go ahead to upgrade all of these projects in an easy way? Forget "easy", but this is one way of doing it.</p>

### Change project target framework

This is the first hurdle. You need to right-click on each property in the solution, select properties and then change the target framework in the window that opens.

When you click save, you've basically set the `targetFramework` setting of the `compilation` node in the `web.config` file.

{% highlight xml %}
<system.web>
  <compilation targetFramework="4.8" />
</system.web>
{% endhighlight %}

Can't you do a search-replace? I've seen many people recommend against it and I've also noticed that for several projects more changes were made than just changes to this setting.

If anyone knows of a easier but still safe way to do this, maybe through a PowerShell script or similar, please let me know.

### Changing runtime target framework

You would think that changing the target framework for each project is enough to make your solution run a later .NET version. But that's not necessarily the case. The `compilation` node is [extrapolated into several different settings](https://devblogs.microsoft.com/aspnet/all-about-httpruntime-targetframework/) of which `httpRuntime` is one of them. If you've specified `httpRuntime` yourself, then you're overriding the default value and you'll prohibit the use of the target framework you've just completed.

{% highlight xml %}
<system.web>
  <compilation targetFramework="4.8" />
  <httpRuntime targetFramework="4.6" />
</system.web>
{% endhighlight %}

The `httpRuntime` is what version of .NET Framework your code will be running and it can be kept lower on purpose to make your code backwards compatible. But to go ahead with the upgrade, you need to manually search through your code for any instances of this setting and then, either remove them so the default upgraded value will be used, or upgrade it manually to the new version. This is usually an easy task since even a big solutions seldom consists of too many runnable projects, most of them are some kind of libraries or test projects.

### Upgrading NuGet packages

Up until now, we've focused on the `web.config` file. But there's also a lot in the `packages.config` file that relates to the .NET version being targetted. A typical such file can look like this:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="AutoMapper" version="____" targetFramework="net46" />
  <package id="Unity" version="____" targetFramework="net46" />
  <package id="WindowsAzure.ServiceBus" version="____" targetFramework="net46" />
</packages>
{% endhighlight %}

You could easily just do a search and replace for all the targetFramework properties. But the problem with this approach is that you have no control over dependencies and issues this might bring.

Luckily for us, there is a [NuGet upgrade tool](https://docs.microsoft.com/en-us/nuget/consume-packages/reinstalling-and-updating-packages) to use for these occasions. What this tool is doing is that it removes and re-installes every single NuGet package in your solution, but this time with the correct .NET Framework target version.

This command can be executed inside the Package Manager Console inside Visual Studio. Running it like this will update the whole solution. Remember to add `-reinstall` at the end or else it'll upgrade each package to the latest possible version during update - something that we probably don't want.

{% highlight csharp linenos %}
Update-Package –reinstall
{% endhighlight %}

Running this tool tock me, in the end, over an hour and a half and resulted in more than 1550 changed files. Not really a success! 200 changes related to upgraded packages.config files that now had the correct .NET version. 



But there were also a lot of other changes that I manually had to go through and approve of.