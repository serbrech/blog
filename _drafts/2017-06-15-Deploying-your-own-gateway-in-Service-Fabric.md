---
layout: post
title: Deploying your own gateway in Service Fabric
date: '2017-06-15T08:15:00+01:00'
tags: microservices, servicefabric
image: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
image2: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
---

In my previous article [Hosting a web application in Service Fabric]({{ site.baseurl }}{% post_url 2016-11-01-Hosting-a-web-application-in-Service-Fabric %}), we deployed a Guest Executable and exposed the application using the built-in `Service Fabric Reverse Proxy`. This works well, and it gives you an out of the box solution to reach into your cluster, from outside of it.

![Exposing the Reverse Proxy through the load balancer exposes the services inside your cluster]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/external-communication.png)

Unfortunately, the built-in gateway does not give you any control over which application will be exposed, nor does it allow you to rewrite urls. In practice, it means that if you enable the reverse proxy, and expose it to the outside world, *everything* endpoints in your cluster are suddenly exposed. 

[The documentation](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reverseproxy) makes quite this clear : 

![Warning! When you configure the reverse proxy's port in Load Balancer, all microservices in the cluster that expose an HTTP endpoint are addressable from outside the cluster.]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/Warning.PNG)

Of course, we do not want this. So how do we expose our http endpoints? You need deploy your own gateway. It sounds like a lot of work, but bear with me, it's actually very simple. We need the same sort of behaviour as the reverse proxy, only with a little bit more control.

Fortunately, we live in a beautiful open source world and this problem has already been solved. The package [`C3.ServiceFabric.HttpServiceGateway`](https://www.nuget.org/packages/C3.ServiceFabric.HttpServiceGateway/1.0.0-rtm-92) is exactly what we need and it is pretty well done. It is an asp.net middleware that routes incoming requests to internal services.
Here is how to make it work in your cluster.

For this to work, you will need 