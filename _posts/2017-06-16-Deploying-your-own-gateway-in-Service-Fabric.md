---
layout: post
title: Deploying your own gateway in Service Fabric
date: '2017-06-16T13:15:00+01:00'
tags: microservices, servicefabric
image: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
image2: "/blog/assets/article_images/ServiceFabric/sfheader.jpg"
---
#Service Fabric reverse proxy

In my previous article [Hosting a web application in Service Fabric]({{ site.baseurl }}{% post_url 2016-11-01-Hosting-a-web-application-in-Service-Fabric %}), we deployed an OWIN hosted website as a Guest Executable and accessed the application using the built-in `Service Fabric Reverse Proxy`. This works, and it gives you an out of the box solution to reach into your cluster from outside of it : 

![Exposing the Reverse Proxy through the load balancer exposes the services inside your cluster]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/external-communication.png)

Unfortunately, the built-in Reverse Proxy does not give you any control over which application will be exposed, nor does it allow you to rewrite urls. It means that if you enable the reverse proxy and expose it to the outside world, *every single* endpoints in your cluster will be suddenly exposed. 

[The documentation](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reverseproxy) makes very this clear : 

![Warning! When you configure the reverse proxy's port in Load Balancer, all microservices in the cluster that expose an HTTP endpoint are addressable from outside the cluster.]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/Warning.PNG)

Of course, we do not want this. So how do we expose our http endpoints selectively? 

In this article, here is what we will do :  

- Use a sample project with a single default aspnet web application deployed in Service Fabric described below  
- Configure and deploy our own gateway  
- Handle the proxy rewrite in our application, and setup the azure load balancer to point at the new gateway  
- Configure the Azure Load Balancer to point at our new gateway  

#Our sample project

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

#Deploy your own gateway

It sounds like a lot of work, but bear with me, it's actually very simple. We need the same sort of behaviour as the reverse proxy, only with a little bit more control.

Fortunately, we live in the beautiful world of open source and this problem has already been solved. The package [`C3.ServiceFabric.HttpServiceGateway`](https://www.nuget.org/packages/C3.ServiceFabric.HttpServiceGateway/1.0.0-rtm-92) is exactly what we need and it is pretty well done. It is an ASP.NET Core Middleware that routes incoming requests to internal services. You can browse the code and read the extensive documentation on github : https://github.com/c3-ls/ServiceFabric-Http
Here is how to set this up in your cluster :

We are going to add a new empty aspnet core service to our Application.

![add default asp.net core stateless service]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/add-Gateway.PNG)

![Choose the empty template]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/add-Gateway2.PNG)

Choose the empty template, this is only going to run our middleware.
This will create a project with a `WebListenerCommunicationListener` configured with a `WebHost` and a `Startup` class that we will look into in a minute.

Let's configure our Endpoint. You will need to know where is our gateway, so we give it a specific port.
In the ServiceManifest, set the port of the Endpoint :
{% highlight xml %}
  <Endpoint Protocol="http" Name="ServiceEndpoint" Type="Input" Port="1664" />
{% endhighlight %}

The gateway is the entry point to the system and it must stay available. We deploy it on every node.
Modify the `Gateway_InstanceCount` parameter that was added in the ApplicationParameters. Make sure that your production deployment has a value of `-1`.  

{% highlight xml %}
  <Parameter Name="Gateway_InstanceCount" Value="-1"/>
{% endhighlight %}

Note that if you deploy to your local cluster, you will not be able to have more than one instance of the service running on the same port. For local dev cluster, either leave this to 1, or let the port be dynamic.  

We now add the nuget package [C3.ServiceFabric.HttpServiceGateway](https://www.nuget.org/packages/C3.ServiceFabric.HttpServiceGateway) to the gateway.  
In the `Startup.cs` file, we need to enable it :  
{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddServiceFabricHttpCommunication();
} 
{% endhighlight %}

And we configure the middleware :  

{% highlight csharp %}
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

Let's walk through it.
The `RunHttpServiceGateway` method maps the path `/MyAwesomeWebsite` to the root of the `WebService` under the `MyApp` Application.

Now, whenever you hit the gateway on port `1664`, at the path `/MyAwesomeWebsite`, the gateway will use the Naming Service to locate `fabric:/MyApp/WebService` in the cluster and your request will be forwarded to one of the instances.

#Application Urls, Reverse Proxy and X-Forwarded-* headers

When you put your application behind a proxy, the request url might not match the resource url. This means your application will not be able to construct the relative urls.  
For example when I navigate to `/MyAwesomeWebsite`, my website thinks I am on `/`, because that is where the request was forwarded to by the gateway.  
If I have a relative a link on the page to `/about` the browser will try to reach `http://<gateway>/about` instead of `http://<gateway>/MyAwesomeWebsite/about`.  
Fortunately, there are commonly used headers to cicumvent this issue: `X-Forwarded-*`. When the proxy forwards a request, it copies the original information necessary to rebuild the urls in these special headers.
And because it's a common use case, there is a middleware for it : [UseForwardedHeaders](https://github.com/aspnet/BasicMiddleware/blob/dev/src/Microsoft.AspNetCore.HttpOverrides/ForwardedHeadersExtensions.cs).  
It is not very well documented, but it is described in [the documentation when hosting asp.net behind nginx](https://docs.microsoft.com/en-us/aspnet/core/publishing/linuxproduction#configure-a-reverse-proxy-server), and it can be used withany proxy, as long as the headers are passed along.  
`C3.ServiceFabric.HttpServiceGateway` is sending these headers. Unfortunately, the `ForwardedHeaders` middleware does not handle `X-Forwarded-PathBase` so we need to handle this one ourselves. It's not too hard though. Here is what is needed in the WebService `Startup.cs` to handle the proxy headers : 

{% highlight csharp %}
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    ForwardedHeadersOptions options = new ForwardedHeadersOptions
    {
        ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedHost  
    };

    //either fill in the known Networks and KnownProxies appropriately, or clear them to allow any.
    options.KnownNetworks.Clear();
    options.KnownProxies.Clear();

    app.UseForwardedHeaders(options);

    //Handle X-Forwarded-PathBase and set the ambient context property. After this, '~' will resolve to the original pathbase from the gateway.
    const string XForwardedPathBase = "X-Forwarded-PathBase";
    app.Use((context, next) =>
    {
        if (context.Request.Headers.TryGetValue(XForwardedPathBase, out StringValues pathBase))
        {
            context.Request.PathBase = new PathString(pathBase);
        }
        return next();
    });
    
    ...

    app.UseStaticFiles();
    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });

}
{% endhighlight %} 

#Configure the Azure Load Balancer

Of course, nobody wants to access your website on port `1664`. Next, we need set up the load balancer to point at our newly deployed gateway.
On a fresh cluster deployed on azure ([in one powershell line!](https://noelbundick.com/2017/05/12/Service-Fabric-Cluster-Quickstart/)), 
there is an existing `AppPortLBRule1` for port 80. Edit it, and change the backend port to point at your gateway port : `1664` in my case.  

![Point the Load Balancer to your gateway]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/Lb.PNG){:height="600px"}

Note that the Load Balancer defines a `Health Probe`. If the health probe does not succeed, the load balancer will assume that the backend pool is not healthy, and it will not redirect your request. Here is a probe that will succeed.  

![Change the Probe to check an existing enpoint]({{ site.baseurl }}/assets/article_images/2017-06-15-Deploying-gateway/Probe.PNG){:height="600px"} 

And Voilà!  
 
I can now reach the website on `http://[FQDN]/MyAwesomeWebsite` and all relative links and resources work!
I hope this was useful. Do not hesitate to reach out in the comments below, or tweet at me: [@serbrech](https://twitter.com/serbrech/)


 

