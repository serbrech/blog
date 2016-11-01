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

Set build output to Console Application : 
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

Then you can do Add New Project, search for Service Fabric, and add a Guest Exectuable Service Fabric project.
This will keep the xml files that describe your executable within the service fabric files, instead of having to add them in your project. Note that the project is basically a set of XML files, a link to the binaries of your app, and a powershell script to deploy it to the cluster. So with some work, you could make this an infrastructure concern, and not even add this project to the solution.
anyway, the procedure is described here : https://azure.microsoft.com/en-us/documentation/articles/service-fabric-deploy-existing-app/  

I will focus on the tweaks I made to get it all work nicely.

1. Set your project's Platform Target x64 (Project Properties > Build > Platform Target). Service Fabric supports only x64.  

2. Chose link for the Code Package Behaviour on project creation (it's the default).  
In my case, linking did not work recursively. So I edited the SF csproj according to [this Stackoverflow answer](http://stackoverflow.com/a/11808911/156415) : 

  {% highlight xml %}
     <Content Include="..\Web\bin\**\*.*"> 
          <Link>ApplicationPackageRoot\ElectronicConsentPkg\Code\%(RecursiveDir)%(FileName)%(Extension)</Link>
     </Content> 
  {% endhighlight %}  

3. ApplicationManifest  
Nothing special here, except the ApplicationTypeName attribute.  
     {% highlight xml %}
     <ApplicationManifest ApplicationTypeName="MyApp" ...>
     {% endhighlight %}
     
This value is important as it defines the id of your application (and part of your service) in Service Fabric service discovery system. It also ends up on the url of your service. more on that below.  

4. ServiceManifest   
That one is more interesting. Here is mine : 
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

That declares a Stateless service. The entrypoint is our Web.exe binary. we set the working directory to Codebase. that's our binary folder in the package, and we pass some arguments.
We define an http endpoint that listens on port 3000 (same as we passed as argument to our app), and a pathSuffix that I am going to explain in the last part of this post.
From there, you have all the elements. if you have a cluster running locally, you can right-click-publish... shivers...
Once it is deployed, navigate to `http://localhost:3000/Myapp/Web` and TADA your app is there. Beautiful isn't it?

## The last step: play nice with the Reverse Proxy

Amazing, we are hosting our app in Service Fabric and can access it directly. However, that's not how you want it in production.
What you really want to do is to have multiple instances of your web app across the cluster, accessible from the same URL, loadbalanced.
Http endpoints are exposed automatically through ServiceFabric reversee proxy at at `http://fqdn:19081/{AppName}/{EnpointName}/`

So, for us locally, that would be : `http://localhost:19081/Myapp/Web/`
The challenge with reverse proxy, is that internally, your app doesn't know it is accessed from a different location.
If you don't match the rootpath from the proxy to the rootpath of your app, the links in your web application will be broken.

So I let the manifest define the root path of my application, which is based on the Service Fabric convention (Appname/EndpointName).
that's the EntryPoint Argument node :  
{% highlight xml %}
<Arguments>3000 MyApp/Web</Arguments>
{% endhighlight %}  

The application responds on `http://localhost:3000/MyApp/Web`, so we have to tell ServiceFabric that this is where our app is.
that's the role of the Endpoint node :  
{% highlight xml %}
<Endpoint Name="Web" Protocol="http" UriScheme="http" Type="Input" Port="3000" PathSuffix="MyApp/Web"/>
{% endhighlight %}  

The PathSuffix tells ServiceFabric that the entry point of the service is at  /Myapp/Web, which need to match where my app is served (the argument to the application). you want them both to match with the application name and the endpoint name, then you are all set.

With this in place, we can use urls relative to the root of our app, through the proxy and when accessing the service directly. It all works fine as expected.
Just make sure to use the tild syntax in your html : `href="~/Content/..."`  

Passing the port as parameter to my application allows me to setup multiple instances at different port locally. In production with multiple instances, they would not be on the same machine, so it would not be a problem anyway.

## Let's Recap

To host an existing IIS web app on Service Fabric we :  
 
 - Change our Platform Target to x64
 - Made it Self-Hosted on OWIN
 - Add a Service Fabric Guest Executable project to the solution
 - Linked our binaries to it
 - Adjusted the xml metadata
 - Parameterized the application root and the port to play nice with reverse proxy  
 - Match the PathSuffix and the application root, with the ApplicationTypeName and endpoint Name.
 
As a result, our application does not need to be aware of Service Fabric. It can still be run and debugged locally as any other application too. This is very nice. We get all the benefits from the platform, without taking any dependencies on it in any way!  

Notice as well that the Service Fabric project we add is just metadata. We could make this part of our deployment infrastructure, and not even get a Service Fabric related project in our solution. Heck, I think we will end up generating this based on conventions, and not worry about it for each project. Outstanding!

