---
layout: post
title: "Azure Functions with static IP"
date: 2021-03-01
tags: azure functions serverless vnet
twitter-title: "Azure Functions with static IP"
description: ""
image: 2019-11-19-deploying-worker-service-to-kubernetes/cargo-ship.jpg
---

<p class="intro"><span class="dropcap">A</span>t work, we have a customer that needs a custom scaleable service accessible via a static IP. Maybe Azure Functions isn't your first suggestion due to the network requirements, but it is actually possible to setup due to new functionality in Azure.</p>

Azure Functions is mainly known and promoted for it's scaleable only-pay-for-what-you-use model that costs almost nothing to run. But there are two more ways of running these functions:

https://notetoself.tech/2020/11/21/azure-functions-with-a-static-outbound-ip-address/
