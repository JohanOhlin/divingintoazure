---
layout: post
title: "VS Code with custom build box"
date: 2021-02-05
tags: vscode linux
twitter-title: "VS Code with custom build box"
description: ""
image: 2019-11-19-deploying-worker-service-to-kubernetes/cargo-ship.jpg
---

<p class="intro"><span class="dropcap">S</span>tarting from February 17, you won't</p>

In this blob post I'll show you how to create and prepare a Linux VM to use as a build box for VS Code.

### Generate key with password

To secure a VM properly, Microsoft recommends that you use a password protected key rather than just a password. There are different ways of creating keys to use. In this example, I'll show you how to create the key in Azure cloud shell associated with your personal login, but there are other ways to do it also.

If you don't already heave a password protected key, create one with this command in the cloud shell. This will create a 4096 bits key for the user name `azureuser`. Remember this user name because you'll use it later on. You can change this user name to whatever you prefer. This command will ask you for a password. This password will be used whenever this key is being used to make note of it somewhere.

{% highlight bash %}
ssh-keygen \
 -m PEM \
 -t rsa \
 -b 4096 \
 -C "azureuser"
{% endhighlight %}

This key will be stored in the .ssh folder in your cloud account. It'll create two files, one for your private key (`id_rsa`) and one for the public (`is_rsa.pub`).

### Create SSH Key in Azure

Whenever you're creating a new VM with key authentication, the public key will be used as the VMs way to authenticate you. To make it easier to handle the public key, you can create an `SSH Key` in Azure that holds the public key so it's ready when you create a VM.

Search for `SSH Key` in Azure and create a new instance. In the public key section, paste in the public key you just created (from `id_rsa.pub`). To show the content of that file, simply type `cat id_rsa.pub` inside the `.ssh` folder.

### Create VM

If you create the VM in the portal, make sure you select Linux image. For this post, I used `Ubuntu 18.04 LTS`. Also deselect `Availability options` if they are preselected for you.

As size, I'm using a `Standard_B1s` with 1 VCPU and 1 GB of RAM costing me around 9 USD/month. This is actually enough to even create docker builds.

For the authentication, select `SSH public key`. `Username` should be the same as you used when creating the key earlier on. If you created an `SSH Key` from you public key in the previous step, then select `Use existing key stored in Azure` as `SSH public key source`. You'll then be able to select from a list the SSH key.

Make sure SSH port 22 is open so we can connect to the VM later on.

Considering OS disk, you have several options. Running a managed disk will give you several improvements regarding reliability and flexibility. You can dynamically increase the size of the disk (but not decrease) something you can't easily with an unmanaged page-blob disk. A 32 GB managed standard ssd disk will currently cost you around 2.60 USD/month.

Options under `Networking` can be left as default, unless you have specific needs.

Under `Management`, make sure to select `Auto-shutdown` to save a little more on your VM cost.

Click `Review + create` and then `Create`.

If you don't want to pay for a constand static IP then you'll get a new IP each time you restart your VM. A better way of doing this is to assign it a DNS name that you can use when connecting to it. Open the VM once it's created and click on `____`. This name needs to be unique in the region where you have your VM.

#### Create VM with Azure CLI

If you want to create the VM in Azure CLI from the cloud shell, use the following commands. Since a VM creates a number of resources (disk, nic, etc) it's easier if the VM gets it's own resource group to stay in. Then, if you want to delete it you know exactly what items to delete.

The parameter `--generate-ssh-keys` will only generate new keys if you don't have any existing ones. Since we created keys above, they will be used instead and the command will then ask you for the password for the keys.

As described in the section about creating a VM in the portal, it's beneficial to assign a DNS name to your VM. Change the variable `uniquednsname` below to something unique before running the script.

{% highlight bash %}
subscription=d044cb10-2b08-4c81-afa3-823af05ca8b6
location=westeurope
resourceGroup=linuxbox3
vmName=linuxbox3
uniquednsname=linboxtest3
image=UbuntuLTS
username=azureuser

# Create a new resource group

az group create --name $resourceGroup --location $location --subscription $subscription

# Create the VM

az vm create \
 --subscription $subscription \
 --resource-group $resourceGroup \
 --name $vmName \
 --image $image \
 --admin-username $username \
 --generate-ssh-keys \
 --storage-sku StandardSSD_LRS \
 --os-disk-size-gb 30 \
 --size Standard_B1s \
 --public-ip-address-dns-name $uniquednsname

# Enable auto shutdown at 6 pm

az vm auto-shutdown \
 --subscription $subscription \
 --resource-group $resourceGroup \
 --name $vmName \
 --time 1800
{% endhighlight %}

Config details for [disks](https://docs.microsoft.com/en-us/cli/azure/disk?view=azure-cli-latest#az_disk_create), [vm](https://docs.microsoft.com/en-us/cli/azure/vm?view=azure-cli-latest#az_vm_create) and [vm sizes](https://docs.microsoft.com/en-us/cli/azure/vm?view=azure-cli-latest#az_vm_create-optional-parameters).

The response from creating the VM will look like this

{% highlight json %}
{
"fqdns": "linboxtest.westeurope.cloudapp.azure.com",
"id": "/subscriptions/d044cb10-2b08-4c81-afa3-823af05ca8b6/resourceGroups/linuxbox2/providers/Microsoft.Compute/virtualMachines/linuxbox2",
"location": "westeurope",
"macAddress": "00-0D-3A-AB-81-66",
"powerState": "VM running",
"privateIpAddress": "10.0.0.4",
"publicIpAddress": "104.214.224.105",
"resourceGroup": "linuxbox",
"zones": ""
}
{% endhighlight %}

Test the connection by using SSH in the cloud console

{% highlight bash %}
ssh azureuser@linboxtest.westeurope.cloudapp.azure.com
{% endhighlight %}

### On new VM

Now it's time to setup the tools you need/want on your build box. In the examples below, I'll build a ASP.NET Core app, build it as a docker container and deploy it azure so these are the tools I'll install here.

#### Install docker

{% highlight bash %}
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update
apt-cache policy docker-ce
sudo apt install -y docker-ce
{% endhighlight %}

Verify docker is running

{% highlight bash %}
sudo systemctl status docker
{% endhighlight %}

#### Install .Net 5 SDK and runtime

{% highlight bash %}
wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update; \
 sudo apt-get install -y apt-transport-https && \
 sudo apt-get update && \
 sudo apt-get install -y dotnet-sdk-5.0
{% endhighlight %}

#### Install Azure CLI

{% highlight bash %}

{% endhighlight %}

# Setup code-spaces

# install puttygen which is part of the putty package

sudo apt-get update
sudo apt-get install putty

# convert id_rsa to private.ppk

puttygen id_rsa -O private -o private.ppk

https://stuarteggerton.com/2020-05-09-2020-05-02-visual-studio-codespaces-self-hosted-md/#:~:text=Setting%20up%20a%20self-hosted%20Ubuntu%2019.01%20Sign%20up,copy%20the%20resource%20id%20id%20including%20the%20quotes
https://docs.microsoft.com/en-us/visualstudio/codespaces/reference/vsonline-cli#installation

curl -L https://aka.ms/install-vso-linux | sudo bash
vso start -i "/subscriptions/c931ad62-62d8-4f25-a0d3-c88aaaaabbbbb/resourceGroups/vso-rg-e315e92/providers/Microsoft.VSOnline/plans/vso-plan-eastus
