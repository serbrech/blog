---
layout: post
title: Hosting a web application in Service Fabric
date: '2016-11-01T08:15:00+01:00'
tags: microservices, servicefabric
image: "/blog/assets/article_images/2016-11-01-hosting/andromeda-galaxy.jpg"
image2: "/blog/assets/article_images/2016-11-01-hosting/andromeda-galaxy.jpg"
---

Service Fabric is a platform that is designed to solve the challenges I exposed [in my previous post on Microservices]({{ site.baseurl }}{% post_url 2016-10-26-What-we-dont-tell-you-about-microservices %}). 
It manages a cluster of machines, and lets you work against it as if it was a single host. You send your app to Service Fabric, and the system will host it somewhere, and make it available at a given endpoint.
Of course, that's highly simplified, it does much, much more. It handles things like service discovery, reverse proxy and load balancing, health monitoring, and scaling, stateful services and more...

So, to assess the platform, I look at what it takes to take an existing small web application, and deploy it on a Service Fabric cluster.
It turns out that it is relatively easy.
My application test is a Nancy web app, deployed to IIS. this is a write up of what I had to do.

## Where do we start?

In Service Fabric, you have 2 main choices when it comes to the type of service you can host : Stateful or Stateless.
A Stateless service is simply an endpoint that doesn't have any state attached to it. An API, a Website, or a gateway for example. Stateless service can have a state, but it will be stored externally in a database for example.
A Stateful service can store the data internally, using Reliable Collections provided by Service Fabric. These look just like your usual collections, but Service Fabric makes them highly available and consistent across your services.
For our example I will use a Stateless Service.

So I want to host my Nancy application as a Stateless Service in the cluster. Ideally, I want my service to be unaware of Service Fabric. 
Service Fabric supports [hosting any executable](https://azure.microsoft.com/en-us/documentation/articles/service-fabric-deploy-existing-app/).

## Step 1 : Make the web application self-hosted

We are going ot use OWIN and it's self-host capabilities to achieve this.  
I start by adding the owin packages to my project : 

     "Microsoft.Owin.Host.HttpListener": "3.0.1"
     "Microsoft.Owin.Hosting": "3.0.1"

We add and OWIN Startup configuration class, a Program.cs file with a `static void Main` and set our project to be a Console App. I will get to why I take parameters in :

{% highlight csharp %}
[assembly: OwinStartup(typeof(Web.Startup))]

namespace Web
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            app.UseNancy();
        }
    }
}
{% endhighlight %}

{% highlight csharp %}
public class Program
{
   public static void Main(params string[] args)
   {

       int port = 1234;
       string appRoot = "";
       // get port and approot from arguments if provided.
       if (args.Any())
       {
           int.TryParse(args[0], out port);
           appRoot = args[1]?.TrimStart('/') ?? "";
       }

       var url = $"http://+:{port}/{appRoot}";
       using (WebApp.Start(url))
       {
           Console.WriteLine("Listening at " + url);
           Console.ReadLine();
       }
   }
}
{% endhighlight %}

Set the build output to `Console Application` : 
![Console Application](/blog/assets/article_images/2016-11-01-hosting/console.PNG)


When we self-host, [we have to tell our web framework where to find the files](http://stackoverflow.com/questions/24571258/how-do-you-resolve-a-virtual-path-to-a-file-under-an-owin-host) (views, content, etc..).  
With Nancy, we need to configure a RootPathProvider :  
{% highlight csharp %}
public class CustomRootPathProvider : IRootPathProvider
{
     public string GetRootPath()
     {
          return Directory.GetCurrentDirectory();
     }
}

public class Bootstrapper : DefaultNancyBootstrapper
{
    protected override IRootPathProvider RootPathProvider {
        get { return new CustomRootPathProvider(); }
    }
    ...
}
{% endhighlight %}

That's it, the app is now self-hosted. you can start it as a console app by running F5 and open your browser at `http://localhost:1234`.

The next step is to host this executable in ServiceFabric.

## Step 2 : Guest Executable in Service Fabric

For this, the Service Fabric SDK with VS tooling needs to be installed. the easiest way is by using the [Web Platform Installer](https://www.microsoft.com/en-us/download/details.aspx?id=6164)

I like to keep my code agnostic of the infrastructure. This is why I choose the guest executable. 
If you have the SDK installed, you can do Add New Project, search for Service Fabric, and add a Guest Exectuable Service Fabric project to your solution.
This will keep the xml files that describe your executable (ServiceManifext.xml) within the service fabric project, instead of having to mix them in your web project. Note that the Service Fabric project is basically a set of XML files, a link to the binaries of your app, and a powershell script to deploy it to the cluster. So with some work, you could make this an infrastructure concern, and not even add this project to the solution.
Anyway, the procedure is well described here : [Deploy a guest executable to Service Fabric](https://azure.microsoft.com/en-us/documentation/articles/service-fabric-deploy-existing-app/)  

I will focus on the tweaks I made to get it all work nicely.

1. Set your project's Platform Target x64 (Project Properties > Build > Platform Target). Service Fabric supports only x64.  

2. Choose link for the Code Package Behaviour on project creation (it's the default).  
In my case, linking did not work recursively. So I edited the SF csproj according to [this Stackoverflow answer](http://stackoverflow.com/a/11808911/156415). This will ensure that subfolders are also included (i.e: content, js, views, etc..). :  

          <Content Include="..\Web\bin\**\*.*"> 
               <Link>ApplicationPackageRoot\MyAppPkg\Code\%(RecursiveDir)%(FileName)%(Extension)</Link>
          </Content>

3. ApplicationManifest  
An application in Service fabric can be composed of multiple services and endpoints. It is your deployment unit.
The ApplicationTypeName value is important as it defines the id of your application (and of your service) in Service Fabric service discovery system. It also ends up being part of the url to your service. More on that below.  

          <ApplicationManifest ApplicationTypeName="MyApp" ...>

4. ServiceManifest   
This one is more interesting. It will describe the endpoints to reach your system. Here what I ended up with, I will explain below :  
          {% highlight xml %}
              <ServiceManifest Name="MyAppPkg"
                           Version="1.0.1"
                           xmlns="http://schemas.microsoft.com/2011/01/fabric"
                           xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
                 <ServiceTypes>
                   <!-- This is the name of your ServiceType. 
                        The UseImplicitHost attribute indicates this is a guest executable service. -->
                   <StatelessServiceType ServiceTypeName="MyAppWeb" UseImplicitHost="true" />
                 </ServiceTypes>

                 <!-- Code package is your service executable. -->
                 <CodePackage Name="Code" Version="1.0.0">
                   <EntryPoint>
                     <ExeHost>
                       <Program>Web.exe</Program>
                       <Arguments>3000 MyApp/Web</Arguments>
                       <WorkingFolder>CodeBase</WorkingFolder>
                     </ExeHost>
                   </EntryPoint>
                 </CodePackage>

                 <ConfigPackage Name="Config" Version="1.0.0" />

                 <Resources>
                   <Endpoints>
                     <Endpoint Name="Web" Protocol="http" UriScheme="http" Type="Input" Port="3000" PathSuffix="MyApp/Web"/>
                   </Endpoints>
                 </Resources>
               </ServiceManifest>
          {% endhighlight %}  

This xml declares a Stateless service. The `EntryPoint` is our `Web.exe` binary. We set the working directory to `Codebase`. Codebase points to our binary folder in the service fabric package. We also pass it some arguments.  
The `Endpoint` node defines where service fabric will find our service.
We define an http endpoint that listens on port 3000, and a PathSuffix that I am going to explain in the last part of this post. I see the Endpoint definition as telling the Service Fabric runtime how to contact our application. if your web app listens on port 3000, you will need to define port 3000 here.
From there, you have all the elements. if you have a cluster running locally, you can [right-click-publish](https://twitter.com/robpearson/status/731865398828630019)... Shivers...
Once it is deployed, navigate to `http://localhost:3000/Myapp/Web` and TADA your app is there. Beautiful isn't it?

## The last step: play nice with the Reverse Proxy

Amazing, we are hosting our app in Service Fabric and can access it directly. However, that's not how you want it in production.
What you really want to do is to have multiple instances of your web app across the cluster, accessible from the same URL, loadbalanced.
Http endpoints are exposed automatically through ServiceFabric reversee proxy at at `http://fqdn:19081/{AppName}/{EnpointName}/`

So, for us locally, that would be : `http://localhost:19081/Myapp/Web/`
The challenge with reverse proxy, is that internally, your app doesn't know it is accessed from a different location.
**If you don't match the rootpath from the proxy to the rootpath of your app, the links in your web application will be broken.**

We are going to let the manifest define the root path of my application, which is based on the Service Fabric convention (AppName/EndpointName).  
This is our the EntryPoint Argument node :  
{% highlight xml %}
<Arguments>3000 MyApp/Web</Arguments>
{% endhighlight %}  

The application responds on `http://localhost:3000/MyApp/Web`. Now we have to tell ServiceFabric that this is where our app is.
We do this in the he Endpoint node :  
{% highlight xml %}
<Endpoint Name="Web" Protocol="http" UriScheme="http" Type="Input" Port="3000" PathSuffix="MyApp/Web"/>
{% endhighlight %}  

The PathSuffix tells ServiceFabric that the entry point of the service is at `/Myapp/Web`. This needs to match where my app is served (the argument to the executable). We want them both to match with the ApplicationName and the endpoint Name, then we are all set.

With this in place, we can use urls relative to the root of our app, through the proxy and when accessing the service directly. It all works fine as expected.
Just make sure to use the tild syntax in your html : `href="~/Content/..."`  

Passing the port as parameter to our application allows us to setup multiple instances at different port locally. In production, multiple instances would not be on the same machine, so it would not be a problem anyway.

## Let's Recap

To host an existing IIS web app on Service Fabric we :  
 
 - Changed our Platform Target to x64
 - Made it Self-Hosted on OWIN
 - Added a Service Fabric Guest Executable project to the solution
 - Linked our binaries to it
 - Adjusted the xml metadata
 - Parameterized the application root and the port to play nice with reverse proxy  
 - Matched the PathSuffix and the application root, with the ApplicationTypeName and endpoint Name
 
As a result, our application does not need to be aware of Service Fabric. It can still be run and debugged locally as any other application too. This is very nice. We get all the benefits from the platform, without taking any dependencies on it in any way!  

Notice as well that the Service Fabric project we add is just metadata. We could make this part of our deployment infrastructure, and not even get a Service Fabric related project in our solution. Heck, I think we will end up generating this based on conventions, and not worry about it for each project. Outstanding!

