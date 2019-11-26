---
layout: post
title: "Develop with CosmosDB in Docker"
date: 2019-07-05
tags: asp.net-core .net-core cosmosdb docker
description: "Micro services are increasingly more popular in new development project. In this post I'll show you how to connect to a CosmosDB emulator from your docker app."
---

<p class="intro"><span class="dropcap">M</span>icro services are increasingly more popular in new development project. In this post I'll show you how to connect to a CosmosDB emulator from your docker app.</p>

### CosmosDB emulator on host

The CosmosDB emulator is locked down to localhost by default. To allow calls from within the docker cluster you need to open up for network access. This is done by adding the `AllowNetworkAccess` parameter at startup. You also need to specify a new access key. The key needs to be a Base64 encoded 64 character long string, like [this one](https://www.base64encode.org/enc/abcdefghijklmnopqrstabcdefghijklmnopqrstabcdefghijklmnopqrstabcd/).

A custom startup script for the CosmosDB emulator could look like this:

{% highlight bash linenos %}
"c:\Program Files\Azure Cosmos DB Emulator\CosmosDB.Emulator.exe" /AllowNetworkAccess /Key=YWJjZGVmZ2hpamtsbW5vcHFyc3RhYmNkZWZnaGlqa2xtbm9wcXJzdGFiY2RlZmdoaWprbG1ub3BxcnN0YWJjZA==
{% endhighlight %}

To connect to the CosmosDB emulator from your docker app you can't use `localhost:8081` anymore since that refers to your docker instance and not the host machine. Instead, if you're using a newer version of docker, then you can use `https://host.docker.internal:8081` to connect to CosmosDB.

If you have an older version of docker then you'll need to connect by IP which works just as fine.

### CosmosDB emulator as docker container

Another way to run the CosmosDB emulator locally is to run it as a container and Microsoft is providing a prepared solution for this. There are different ways to configure your container and you can find more information about it on the [docker image page](https://hub.docker.com/r/microsoft/azure-cosmosdb-emulator/).

For a basic scenario you can run the following commands to setup a container.

{% highlight bash linenos %}
cmd
set containerName=azure-cosmosdb-emulator
set hostDirectory=%LOCALAPPDATA%\azure-cosmosdb-emulator.hostd
md %hostDirectory% 2>nul
docker run --name %containerName% --memory 2GB --mount "type=bind,source=%hostDirectory%,destination=C:\CosmosDB.Emulator\bind-mount"  --interactive --tty -p 8081:8081 -p 8900:8900 -p 8901:8901 -p 8979:8979 -p 10250:10250 -p 10251:10251 -p 10252:10252 -p 10253:10253 -p 10254:10254 -p 10255:10255 -p 10256:10256 -p 10350:10350 microsoft/azure-cosmosdb-emulator
{% endhighlight %}

This will download an image of around 10Gb in size so make sure you have a good internet connection. A few things to note for this setup:

- It will bind the ports used by CosmosDB from the container to your host machine. This means that you shouldn't try to run the emulator on the host itself.

- CosmosDB emulator will generate a certificate to a local folder. The command binds that local folder to `%LOCALAPPDATA%\azure-cosmosdb-emulator.hostd`. From your host machine you'll be able to access this new certificate together with a powershell script to help you import this certificate.

- A container named `azure-cosmosdb-emulator` will be created. If you try to run the last command a second time you'll get an error. Instead, use the commands `docker start azure-cosmosdb-emulator` and `docker stop azure-cosmosdb-emulator` to handle your container.

#### Browsing to the emulator explorer

Since we linked port 8081 when starting the emulator container earlier on, we can now browse the emulator explorer just like we would when running the emulator on the host machine using the address `https://localhost:8081/_explorer/index.html`.

#### Using the right Windows container version

When I started to build this solution my Windows host only supported Windows containers of version 1803. This turned out to be a problem due to a bug which in some situations set the UTC time to be UTC-8 hours (Redmond time, hmmm?). In my case it caused the CosmosDB emulator (or maybe the client) to fail all calls since it only allows a time skew of 15 minutes. There was apparently quite some written on Internet about this (always nice to know you're not the first one encountering this problem) and the solution was either to change how `docker run` executes the container or to upgrade Windows to higher version. The former solution would prevent me from using Visual Studio Docker tools and wasn't a preferred option. I therefore opted for a Windows upgrade.

The current windows container being used in these examples is `mcr.microsoft.com/dotnet/core/aspnet:2.2-nanoserver-1903` and the time bug has been fixed here.

#### Conclusions

Installing the docker image for CosmosDB Emulator includes a lot of fuss. It works just fine to connect to a standard Windows installation from the docker app so my recommendation is therefore to keep it simple.