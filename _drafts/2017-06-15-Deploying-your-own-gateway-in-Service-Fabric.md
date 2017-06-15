---
layout: post
title: Deploying your own gateway in Service Fabric
date: '2017-06-15T08:15:00+01:00'
tags: microservices, servicefabric
image: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
image2: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
---
#Service Fabric reverse proxy

In my previous article [Hosting a web application in Service Fabric]({{ site.baseurl }}{% post_url 2016-11-01-Hosting-a-web-application-in-Service-Fabric %}), we deployed an OWIN hosted website as a Guest Executable and accessed the application using the built-in `Service Fabric Reverse Proxy`. This works, and it gives you an out of the box solution to reach into your cluster from outside of it.

![Exposing the Reverse Proxy through the load balancer exposes the services inside your cluster]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/external-communication.png)

Unfortunately, the built-in gateway does not give you any control over which application will be exposed, nor does it allow you to rewrite urls. It means that if you enable the reverse proxy and expose it to the outside world, *every single* endpoints in your cluster will be suddenly exposed. 

[The documentation](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reverseproxy) makes very this clear : 

![Warning! When you configure the reverse proxy's port in Load Balancer, all microservices in the cluster that expose an HTTP endpoint are addressable from outside the cluster.]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/Warning.PNG)

Of course, we do not want this. So how do we expose our http endpoints selectively? 

#Our sample project

It sounds like a lot of work, but bear with me, it's actually very simple. We need the same sort of behaviour as the reverse proxy, only with a little bit more control.

Fortunately, we live in the beautiful world of open source and this problem has already been solved. The package [`C3.ServiceFabric.HttpServiceGateway`](https://www.nuget.org/packages/C3.ServiceFabric.HttpServiceGateway/1.0.0-rtm-92) is exactly what we need and it is pretty well done. It is an ASP.NET Core Middleware that routes incoming requests to internal services. You can browse the code and read the extensive documentation on github : https://github.com/c3-ls/ServiceFabric-Http
Here is how to set this up in your cluster :

My sample project just contains the default asp.net service. This will represent the service you want to expose. not our gateway. 

![default asp.net core stateless service]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/default-aspnet-service.PNG)

We configure it to run 3 instances. In the Application Parameters, I set the *_InstanceCount Parameter value to 3: 

{% highlight xml %}
  <Parameters>
    <Parameter Name="WebService_InstanceCount" Value="3"/>
  </Parameters>  
{% endhighlight %}

To let Service Fabric select a port, we will remove the `Port` attribute from the service manifest endpoint :

{% highlight xml %}
  <Endpoint Protocol="http" Name="ServiceEndpoint" Type="Input" />
{% endhighlight %}

At this point, if your Reverse proxy is enable, you can reach your website through the built-in Reverse Proxy at http://<FQDN>:19081/<ApplicationName>/<ServiceName>

Deploying on a local development cluster in my case, that is `http://localhost:19081/MyApp/WebService/`

As warned earlier, that's not what you want to expose to the outside world.

#Let us deploy our own gateway.

We are going to add a new aspnet service to our cluster. For this example, I will add it to the same application, but you could just as well create it as a separate app.

![add default asp.net core stateless service]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/add-Gateway.PNG)

![Choose the empty template]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/add-Gateway2.PNG)

Choose the empty template, this is just going to host our middleware.
This will create a project with a WebListenerCommunicationListener configured with a WebHost and a Startup class that we will look into in a minute.

First let's configure our Endpoint. You will need to reach the gateway directly, so give it a specific port.
In the ServiceManifest, make sure the port of the Endpoint is set.
{% highlight xml %}
  <Endpoint Protocol="http" Name="ServiceEndpoint" Type="Input" Port="1664" />
{% endhighlight %}

Since this gateway is the entry point to your system, you want it available, we deploy it on every node.
A parameter was added to the xml files in the ApplicationParameters. Make sure that your production deployment has a value of -1.
Note that if you deploy to your local cluster, you will not be able to have more than one instance of the service running on the same port. For local dev cluster, either leave this to 1, or let the port be dynamic.    
{% highlight xml %}
  <Parameter Name="Gateway_InstanceCount" Value="-1"/>
{% endhighlight %}

We now add the nuget package [C3.ServiceFabric.HttpServiceGateway](https://www.nuget.org/packages/C3.ServiceFabric.HttpServiceGateway) to the gateway.
In the Startup.cs file, we need to enable it : 

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddServiceFabricHttpCommunication();
} 
{% endhighlight %}

and we configure the middleware : 

{% highlight csharp}
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole();
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.RunHttpServiceGateway("/MyAwesomeWebsite", "fabric:/MyApp/WebService");
    
    // catch-all
    app.Run(async context =>
    {
        var logger = loggerFactory.CreateLogger("Catch-All");
        logger.LogWarning("No endpoint found for request {path}", context.Request.Path + context.Request.QueryString);

        context.Response.StatusCode = (int)System.Net.HttpStatusCode.NotFound;
        await context.Response.WriteAsync("Not Found");
    });
}
{% endhighlight %}

This is quite self-explainatory, but let's walk through it.
The `RunHttpServiceGateway` method maps the path `/MyAwesomeWebsite` to the root of the WebService under the MyApp Application.

Now, whenever you will hit the gateway on port 1664 in my case, at te path /MyAwesomeWebsite, you will be served the the WebService.
Of course, nobody wants to access your website on port 1664, so next, we will we set up he LoadBalancer!


