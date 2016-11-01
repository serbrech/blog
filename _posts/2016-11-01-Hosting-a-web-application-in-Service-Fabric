---
layout: post
title: Hosting a web application in Service Fabric
date: '2016-11-01T08:15:00+01:00'
tags: microservices, service fabric
image: "/blog/assets/article_images/2016-10-26-what-we-dont-tell-you/Menhirs_carnac.jpg"
image2: "/blog/assets/article_images/2016-10-26-what-we-dont-tell-you/Menhirs_carnac.jpg"
---

Service Fabric is a platform that is designed to solve the challenges I exposed in my previous post on Microservices.
It manages a cluster of machines, and let you work against it as if it was a single host. You send your app to Service Fabric, and the system will host it somewhere, and make it available at a given endpoint.
It handles things like service discovery, revers proxy and load balancing, health monitoring, and more...

So, to assess the platform, I'm starting by looking at what it takes to take an existing small web application, and deploy it on a Service Fabric cluster.
It turns out that it is relatively easy.
My application was a Nancy web app, simply deployed to IIS.

## Where to start.
In Service Fabric, you have 2 main choices when it comes to the type of service you can host : Stateful or Stateless.
A Stateless service is simply an endpoint that doesn't have any state attached to it. An API, a Website, or a gateway for example. Stateless service can have a state, but it will be stored externally in a database for example.
A Stateful service can store the data internally, using Reliable Collections provided by Service Fabric. These look just like your usual collections, but Service Fabric makes them highly available and consistent across your services.
For our example I will use a Stateless Service.

So I want to host my Nancy application as a Stateless Service in the cluster. Ideally, I want my service to be unaware of Service Fabric. 
Service Fabric supports (hosting any executable)[https://azure.microsoft.com/en-us/documentation/articles/service-fabric-deploy-existing-app/].

## Step 1 : Make the web application self hosted

We are going ot use OWIN and it's self-host capabilities to achieve this.

I start by adding the owin packages to my project : 

     "Microsoft.Owin.Host.HttpListener": "3.0.1"
     "Microsoft.Owin.Hosting": "3.0.1"







Service Fabric is, you should start here : (Service Fabric Overview)[https://azure.microsoft.com/en-us/documentation/articles/service-fabric-overview/]  
