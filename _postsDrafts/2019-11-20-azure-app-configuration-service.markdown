---
layout: post
title: "Working with the Azure App Configuration service"
date: 2019-10-05
tags: c# azure app-configuration
---

<p class="intro"><span class="dropcap">W</span>hen it comes to configuring app settings in Azure, there's always been a gap in the tools available. Each app service had its own app settings section but it was difficult to get a good overview. Variables could be injected through the CI/CD pipeline, but even there it wasn't easy to manage. </p>

The App Configuration service is currently in public preview and might have changed slightly by the time you read this, but the main parts should be the same.

### One per environment

It should be mentioned that Microsoft recommends you to not use labels to separate different environments but to rather create different App Configuration instances for that purpose. The reason is the ability to limit access for users in a more fine grained way.

### Import / export

### Compare between environments

### Using in development

Microsoft has good documentation on how to use it for an ASP.NET Core application as well as an 