---
layout: post
title: "Deploying a worker service to Kubernetes"
date: 2019-10-22
tags: c# azure .net-core worker-service kubernetes
---

<p class="intro"><span class="dropcap">O</span>nce you have the worker service configured with health checks it's time to get it deployed. In this article I'll show you how to deploy the service to AKS (Kubernetes) using ACR and Helm.</p>

### Create docker image

If you created your project with docker support, then you should have a docker file in your project root. If not, then you can create a new one by right-clicking the project and selecting `Add` -> `Docker Support`. The docker might differ, depending on what version of Visual Studio you're using. But it should look something like this.

{% highlight docker linenos %}
FROM mcr.microsoft.com/dotnet/core/aspnet:3.0-buster-slim AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/core/sdk:3.0-buster AS build
WORKDIR /src
COPY ["SimpleWebApp/SimpleWebApp.csproj", "SimpleWebApp/"]
RUN dotnet restore "SimpleWebApp/SimpleWebApp.csproj"
COPY . .
WORKDIR "/src/SimpleWebApp"
RUN dotnet build "SimpleWebApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SimpleWebApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SimpleWebApp.dll"]
{% endhighlight %}

### Creating the Helm chart 

Helm is a package manager for Kubernetes that abstracts away many of the complexities of a deployment using the native yaml files you otherwise would have to use. I'll be using Helm 2 here sinc that's the current stable version, but if you have the chance then use Helm 3 instead since it isn't dependent on Tiller and the security problems that come with that.

Right-click on your `project` and select `Add` -> `Container Orchestration Support`. In the dialog that comes up, select `Kubernetes/Helm`. This will create a chart (package) for your project and setup a connection to Azure Dev Spaces (a really cool feature that I'll write more about another time).

If you don't have the `Kubernetes/Helm` option then you can create a Helm chart manually this way:

- Create a folder named `charts`.
- Inside it, run the following command.

{% highlight csharp linenos %}
helm create InvoiceWorker
{% endhighlight %}

This command will create a chart (package) named InvoiceWorker that we'll configure to work with our docker image. This Helm chart will almost look the same as the default one that Visual Studio creates. 

When the Helm chart is created, it creates templates for deployment (pods), service and ingress. However, the worker service only needs a deployment since there's no inbound traffic. The health check endpoint on the worker service is accessed directly on each pod by Kubernetes and for that, the deployment is just enough. You can therefore delete the files `ingress.yaml` and `service.yaml` from your chart.

In the file `values.yaml` you can remove the sections `service:` and `ingress:` since they are neither used any more.

### Kubernetes deployment


### Kubernetes health checks

Kubernetes has two functionalities for checking the health of a pod:
<ul>
<li><b>Readiness</b> - checks if pod has started up and is ready to receive requests. Since we have a worker service, we don't need to think about this one.</li>
<li><b>Liveness</b> - checks if pod is alive and working as it should. This is the check we can hook into.</li>
</ul>

{% highlight csharp linenos %}

{% endhighlight %}



https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes


https://www.quora.com/What-is-p99-latency

https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes