---
layout: post
title: "Deploying a worker service to Kubernetes"
date: 2019-11-19
tags: c# azure .net-core worker-service kubernetes
twitter-title: "Deploying a #netcore worker service to #Kubernetes"
description: "Once you have the worker service configured with health checks it's time to get it deployed. In this article I'll show you how to deploy the service to AKS (Kubernetes) using ACR and Helm."
image: 2019-11-19-deploying-worker-service-to-kubernetes/cargo-ship.jpg
---

<p class="intro"><span class="dropcap">O</span>nce you have the worker service configured with health checks it's time to get it deployed. In this article I'll show you how to deploy the service to AKS (Kubernetes) using ACR and Helm.</p>

This article is based on a [previous article]({{ site.baseurl }}{% post_url 2019-11-07-worker-service-in-kubernetes-with-health-checks %}) where we created a Worker Service in .NET Core 3 with health check endpoints enabled.

In a real world scenario you would set up complete CI/CD pipelines doing some of these steps for you. However, in this article I'll do all the steps manually to show a bit more in detail how it's being done.

### Create docker image

If you created your project with docker support, then you should have a docker file in your project root. If not, then you can create a new one by right-clicking the project and selecting `Add` -> `Docker Support`. You'll be asked what `Target OS` to create the dockerfile for, either Linux or Windows. This is not the OS on your machine but rather what OS the container will be running on and you can use both Linux and Windows on a Windows computer.

The dockerfile might differ depending on what version of Visual Studio you're using. But it should look something like this.

{% highlight docker %}
FROM mcr.microsoft.com/dotnet/core/aspnet:3.0-buster-slim AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/core/sdk:3.0-buster AS build
WORKDIR /src
COPY ["WorkerServiceWithHealthChecks/WorkerServiceWithHealthChecks.csproj", "WorkerServiceWithHealthChecks/"]
RUN dotnet restore "WorkerServiceWithHealthChecks/WorkerServiceWithHealthChecks.csproj"
COPY . .
WORKDIR "/src/WorkerServiceWithHealthChecks"
RUN dotnet build "WorkerServiceWithHealthChecks.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "WorkerServiceWithHealthChecks.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WorkerServiceWithHealthChecks.dll"]
{% endhighlight %}

To build a docker image from your dockerfile you need to have docker running on the build machine, which is the local dev machine for me. I have [Docker Desktop](https://www.docker.com/products/docker-desktop) installed, running Linux containers. 

The `FROM` line in the dockerfile above tells what source to use. `aspnet:3.0-buster-slim` is a Debian Linux source containing ASP.NET 3.0. More possible sources can be found [here](https://hub.docker.com/_/microsoft-dotnet-core-aspnet/).

In Visual Studio, right-click on a dockerfile and select `Build Docker image` to build it. You can also build a Docker image using the command line

{% highlight bash %}
docker build --rm -f "Dockerfile" -t workerservicewithhealthchecks:latest .
{% endhighlight %}

### Push docker image to a container registry

A container registry is like a code repository on internet, but for container images. These images can then easily be pulled later on by different resources that need them. In this example, I'll use Azure Container Registry (ACR) to store the built images.

For the purpose of this article, I've setup a basic container registry named `blogacrtest`, accessible through the URI `blogacrtest.azurecr.io`. 

To push an image to ACR from your command prompt you need to first have [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed. When it's installed you can login to ACR this way:

{% highlight bash %}
az login
az acr login -n blogacrtest
{% endhighlight %}

If you have created an ACR instance separately from the AKS instance then they need to be linked together for AKS to have permissions to pull images. There are different ways of doing it. One of the newer options is to use the update command for AKS. It's currently in preview mode so you need to enable preview features before you can use it. This might not be an option for you if you're running your cluster in production. This way of linking only works if they are in the same subscription.

{% highlight bash %}
az aks update -n myAKSCluster -g myResourceGroup --attach-acr blogacrtest
{% endhighlight %}

Run the following command to view all the existing images you've created. Your recent image `myworkerservice` should be there in the list with the tag `latest`.

{% highlight bash %}
docker image list
{% endhighlight %}

We need to create a tag target for the new container registry that refers to the local image.

{% highlight bash %}
docker tag workerservicewithhealthchecks blogacrtest.azurecr.io/workerservicewithhealthchecks:0.0.1
{% endhighlight %}

When we now list all images we'll see how both images point to the same image id.

{% highlight bash %}
REPOSITORY                                             TAG                 IMAGE ID            CREATED             SIZE
blogacrtest.azurecr.io/workerservicewithhealthchecks   0.0.1               59aa0a033cb6        4 minutes ago       210MB
workerservicewithhealthchecks                          latest              59aa0a033cb6        4 minutes ago       210MB
{% endhighlight %}

And then finally we can push the image.

{% highlight bash %}
docker push blogacrtest.azurecr.io/workerservicewithhealthchecks:0.0.1
{% endhighlight %}

This will take some time when we do it for the first time.

{% highlight bash %}
The push refers to repository [blogacrtest.azurecr.io/workerservicewithhealthchecks]
58d48a1xxxxx: Pushed
ce65098xxxxx: Pushed
0a37aeexxxxx: Pushing [================>                                  ]  25.78MB/76.45MB
1c7781dxxxxx: Pushed
00672cbxxxxx: Pushing [===========================>                       ]  23.01MB/41.34MB
2db44bcxxxxx: Pushing [=======>                                           ]  10.32MB/69.21MB
{% endhighlight %}

I won't go into detail how a docker image is built up, but basically it's split up into separate layers, out of which your code only is one of them. The other layers are related to the framework your code is running on. When you make changes to your code and push anew then only that layer needs to be pushed, the others stay as they are. The subsequent pushes will therefore go much quicker, unless you change the source part in your docker file.

In Azure Container Registry it should now look like this.

{%
  include image.html
  url="2019-11-19-deploying-worker-service-to-kubernetes/container-registry.png"
  alt="Docker images in Azure Container Registry after being pushed"
  description=""
%}

### Creating a Helm chart for deployment

[Helm](https://v3.helm.sh/) is a package manager for Kubernetes that abstracts away many of the complexities of a deployment using the native yaml files, something you otherwise would have to use. Helm 3 has recently reached release candidate level so I'll use that version here instead of the stable Helm 2. The main reason is that Helm 3 has ditched the [tiller](https://helm.sh/docs/glossary/#tiller) component that Helm 2 forced you to install in your kubernetes cluster with way to high access rights. Now with that [security issue](https://engineering.bitnami.com/articles/running-helm-in-production.html) gone we should stick to Helm 3.

A [Helm chart](https://helm.sh/docs/developing_charts/#charts) is the package that bundles your docker image file into a deployable unit. Helm has a lot of features to offer, including defining dependencies between several different Helm charts and also, once you've installed a Helm chart in Kubernetes, the ability to rollback to a previous version of your chart.

The official way to create a Helm chart in Visual Studio is still using Helm version 2. Right-click on your `project` and select `Add` -> `Container Orchestration Support`. In the dialog that comes up, select `Kubernetes/Helm`. This will create a chart for your project and setup a connection to Azure Dev Spaces (a really cool feature that I'll write more about another time).

But if you want to use Helm 3 you have to do a bit more manual work. Installing Helm in Windows is done by using chocolatey, but that has currently also only Helm 2 to offer. Keep an eye on [this page](https://chocolatey.org/packages/kubernetes-helm) for what Helm versions being supported.

The current way to install Helm 3 is to download it from the [releases page](https://github.com/helm/helm/releases) and add the folder with the binary to the path environment variable. If you run `helm version` you should see the current client version being printed on screen.

{% highlight bash %}
> helm version
version.BuildInfo{Version:"v3.0.0-rc.3", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.13.4"}
{% endhighlight %}

If you have Helm 2 installed you'll see two lines here, the client and tiller in the cluster. If tiller isn't configured, you'll see an error, as in the example below.

{% highlight bash %}
> helm version
Client: &version.Version{SemVer:"v2.16.0", GitCommit:"...", GitTreeState:"clean"}
Error: could not find tiller
{% endhighlight %}

Inside your code project, create a folder named charts. Inside that folder, run the following command to create a Helm chart for your docker image. It's important to write the chart name in lower case or you'll end up with errors later on during deployment.

{% highlight bash %}
helm create workerservicewithhealthchecks
{% endhighlight %}

When the Helm chart is created, it creates templates for deployment (pods), service and ingress. However, the worker service, that we're trying to deploy here, only needs a deployment since there's no inbound traffic. The health check endpoint on the worker service is accessed directly on each pod by Kubernetes and for that, the deployment is just enough. You can therefore delete the files `ingress.yaml`, `service.yaml` and `serviceaccount.yaml` from your chart.

In the file `values.yaml` you can remove the section `service:`. Also change `ingress` to have `enabled: false` and `serviceAccount:` to have `create: false`. Change the section `image:` to link to our new repository. These sections in the file should then look like this:

{% highlight yaml %}
image:
  repository: blogacrtest.azurecr.io/workerservicewithhealthchecks
  pullPolicy: IfNotPresent

ingress:
  enabled: false

serviceAccount:
  create: false

... 
{% endhighlight %}

In the file `Chart.yaml` we have a setting `appVersion` that needs to reflect the tag we've assigned to the image. In the example above we set the value to `0.0.1` so that's the value we need to change the app version to. Don't confuse this setting with `version` which is the setting of the chart that you're creating. If you change anything in the chart settings, but not in the app code, then you bump `version`. If you make changes in your app code then you bump both `version` and `appVersion`. The `Chart.yaml` file now looks like this:

{% highlight yaml %}
apiVersion: v2
name: workerservicewithhealthchecks
description: Worker Service with Health Checks
type: application
version: 0.0.1
appVersion: 0.0.1
{% endhighlight %}

The file `NOTES.txt` contains information about the new installation that will be written out on the screen after the installation is successful. Since we won't have a service running, we need to make some alternations here. This sample explains how to setup a proxy to view the health endpoint without running a complete kubernetes service and ingress. Just paste this text into the file and save it.

{% highlight yaml %}
{% raw %}
For testing purposes, you can now create a proxy on your local machine to access the service.
Run these commands and then navigate to http://127.0.0.1:8080/health to view the health status endpoint

export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "workerservicewithhealthchecks.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:80
{% endraw %}
{% endhighlight %}

Finally, the chart has installed a test to see that the service is up and running. Since we're not using a service, we can remove the `tests` folder. 

We're almost done with the helm chart, but first we need to tweak the deployment template to work with health checks.

### Kubernetes health checks

In the [previous article]({{ site.baseurl }}{% post_url 2019-11-07-worker-service-in-kubernetes-with-health-checks %}), we created a worker service with health checks enabled. Here, we'll hook it up to the infrastructure so Kubernetes can be made aware of the status of the worker service.

Kubernetes has two functionalities for checking the health of a pod:

- **Readiness** - checks if pod has started up and is ready to receive requests. Since we have a worker service, we don't need to think about this one.
- **Liveness** - checks if pod is alive and working as it should. This is the check we can hook into.

If we open up the `deployment.yaml` file from the Helm chart, we find a section inside the `containers:` looking like this:

{% highlight csharp %}
livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http
{% endhighlight %}

We need to make some changes here, but first we need to take a look at what options we have. There are different ways to make [health checks](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes) in Kubernets:

- [Command](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command) - run a command inside pod to figure out its health
- [HTTP probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request) - a GET call to a specific endpoint
- [TCP probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe) - call to a specific port

In this case, we'll use the HTTP probe since that's what we setup in the previous article. A simple HTTP health check can look like this.

{% highlight csharp %}
livenessProbe:
  httpGet:
    path: /health
    port: 80
{% endhighlight %}

But there are also other [settings](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes) you can use here to configure the probe:

- **initialDelaySeconds** - Number of seconds to wait until starting to check the liveness probe. Default is 0.
- **periodSeconds** - How many seconds between each probe. Default is 10.
- **timeoutSeconds** - Seconds to wait until a timeout is reached. Default is 1.
- **successThreshold** - Number of consecutive successful calls needed after a failed one before it's classified as successful again. Default is 1 and liveness probe needs 1 to work properly.
- **failureThreshold** - Number of consecutive failed calls needed until it's classified as failed and Kubernetes will restart the container. Default is 3.

Many of these values are working fine for us by default, but not all. As described in the previous article, a Worker Service is locking the thread until an async/await is reached. During this time, no response will be given on the health endpoint. The value for `timeoutSeconds X failureThreshold` needs to be set to a value that is safe considering how often the thread will be released for the health endpoint. This makes these settings highly individual per worker service.

Another value you might need to adjust is the `initialDelaySeconds` in case your worker service takes some time to start up.

So in the end, our deployment configuration might look something like this:

{% highlight csharp %}
livenessProbe:
  httpGet:
    path: /health
    port: http
  failureThreshold: 6
  timeoutSeconds: 5
  periodSeconds: 15
  initialDelaySeconds: 30
{% endhighlight %}

The key here is that a Worker Service doesn't need to be as responsive as a Web Service, and thus it's OK if it takes longer time for Kubernetes to find a stuck pod. Try to aim to cover for up to at least the [99th percentile](https://www.quora.com/What-is-p99-latency).

### Kubernetes deployment

We have now completed setting up the Helm chart for the worker service. If we want to test-run the installation we can do so by navigating to the charts folder and then run the following command. The first name is the name of the installation in Kubernetes and the second name is the chart name.

{% highlight bash %}
helm install workerservicewithhealthchecks workerservicewithhealthchecks --dry-run --debug
{% endhighlight %}

If we're happy with the result we can go ahead and install it for real with this command:

{% highlight bash %}
helm install workerservicewithhealthchecks workerservicewithhealthchecks
{% endhighlight %}

The result should then be displayed as:

{% highlight bash %}
Release "workerservicewithhealthchecks" has been upgraded. Happy Helming!
NAME: workerservicewithhealthchecks
LAST DEPLOYED: Wed Nov 20 08:08:55 2019
NAMESPACE: default
STATUS: deployed
REVISION: 5
NOTES:
For testing purposes, you can now create a proxy on your local machine to access the service.
Run these commands and then navigate to http://127.0.0.1:8080/health to view the health status endpoint

export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=workerservicewithhealthchecks,app.kubernetes.io/instance=workerservicewithhealthchecks" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace default port-forward $POD_NAME 8080:80
{% endhighlight %}

If you later on want to make an upgrade to your code you'll need to go through the following steps:

- Change code
- Build new docker image
- Tag docker image with ACR domain and new tag version
- Update `Charts.yaml` with new version and appVersion
- Call `helm upgrade` instead of install, as seen below 

{% highlight bash %}
helm upgrade workerservicewithhealthchecks workerservicewithhealthchecks
{% endhighlight %}

### Following the results in Kubernetes

To view all the installed pods, run `kubectl get pods`. In the result you can see the name of your pod and also how many times Kubernetes has restarted it (remember from the previous article that we deliberately made it crash so Kubernetes would restart it).

{% highlight bash %}
NAME                                             READY   STATUS    RESTARTS   AGE
workerservicewithhealthchecks-6c69799695-gxmgv   1/1     Running   3          51m
{% endhighlight %}

We can now get more detailed information about this pod by running

{% highlight bash %}
kubectl describe pod workerservicewithhealthchecks-6c69799695-gxmgv
{% endhighlight %}

The result is a long list of data, including information about the health checks we have configured, but at the bottom you'll find the recent events, including how Kubernetes has been killing and restarting the pod. In the helm chart, we configured that it would take 6 failed health checks before it should be deemed as bad. That's why the `Unhealthy` warning in the event has 6 times as many events as the event `Killing`.

{% highlight bash %}
Events:
  Type     Reason     Age                  From                               Message
  ----     ------     ----                 ----                               -------
  Normal   Scheduled  34m                  default-scheduler                  Successfully assigned default/workerservicewithhealthchecks-6c69799695-gxmgv to aks-nodepool1-abc-0
  Normal   Pulling    34m                  kubelet, aks-nodepool1-abc-0  Pulling image "ohlin.azurecr.io/workerservicewithhealthchecks:0.0.3"
  Normal   Pulled     34m                  kubelet, aks-nodepool1-abc-0  Successfully pulled image "ohlin.azurecr.io/workerservicewithhealthchecks:0.0.3"
  Warning  Unhealthy  8m6s (x12 over 22m)  kubelet, aks-nodepool1-abc-0  Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    8m6s (x2 over 21m)   kubelet, aks-nodepool1-abc-0  Container workerservicewithhealthchecks failed liveness probe, will be restarted
  Normal   Created    8m5s (x3 over 34m)   kubelet, aks-nodepool1-abc-0  Created container workerservicewithhealthchecks
  Normal   Started    8m5s (x3 over 34m)   kubelet, aks-nodepool1-abc-0  Started container workerservicewithhealthchecks
  Normal   Pulled     8m5s (x2 over 21m)   kubelet, aks-nodepool1-abc-0  Container image "ohlin.azurecr.io/workerservicewithhealthchecks:0.0.3" already present on machine
{% endhighlight %}

Another way to follow the results of how kubernetes changes the status of the pod as it gets restarted, is to list pods with a watch that updates the list as changes occur.

{% highlight bash %}
kubectl get pods --watch
{% endhighlight %}

This will result in a list like this where each restart creates a new line

{% highlight bash %}
NAME                                             READY   STATUS    RESTARTS   AGE
workerservicewithhealthchecks-6c69799695-gxmgv   1/1     Running   421        3d19h
workerservicewithhealthchecks-6c69799695-gxmgv   1/1     Running   422        3d19h
workerservicewithhealthchecks-6c69799695-gxmgv   1/1     Running   423        3d19h
workerservicewithhealthchecks-6c69799695-gxmgv   1/1     Running   424        3d19h
{% endhighlight %}

### Conclusions

In this and the previous article, we've created a worker service in .NET Core 3 that mainly works on a timer but also exposes a health endpoint that Kubernetes uses to restart it when needed. We have not configured any service or ingress objects enabling external access and it's therefore not possible to access the service pod to manually check the health status unless you also setup a proxy using kubectl, but that shouldn't be used in a production environment.

In a real world scenario, you would therefore also need hook up logging to a service like Azure Monitor. This can provide a lot of useful and needed telemetry about how the service is processing the jobs it's expected to do, something we don't know (or care) about, from a liveness point of view.

Despite being slightly more fiddly in the setup, adding a health check to a background service gives you a good way to add automatic restart functionality with health rules that you can define for yourself in the code. The possibility is there - why not use it?