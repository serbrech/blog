---
layout: post
title: Abstract Factory and IoC Container in asp.net MVC
date: '2010-06-26T17:38:00+02:00'
tags:
- Asp.net
- IoC
- MVC
tumblr_url: http://alsagile.com/post/9707604191/mvc-abstract-factory-ioc
---
Sometimes, you want to be able to change the implementation of a component depending on a variable that you don’t control at design time, a user action for example. This will be a lot easier to explain if we use a concrete example. I will present the problem that I faced to illustrate the usage of the method.

Consider a simple search form, but you want to be able, with the same form, to search in different data stores.

You already know that more data stores will be added to the requirements, so you also need a way to extend this it easily. The following solution supposes that you know the basic concepts of dependency injection and how to setup your mvc application to take advantage of it.

I use StructureMap here, but it could of course be swapped by any other IoC container.

Consider that you will have different implementations of this Search action, which will build the query in different ways, and each will make calls to different APIs. You create an interface ISearchManager as follow :

public interface ISearchManager
{
    IEnumerable<ResultItem> Search(string keyword);
}

And the possible implementations :

public class TwitterSearchManager&#160;: ISearchManager
{
    public IEnumerable<ResultItem> Search(string keyword){ //Search using Twitter Api and returns a collection of Tweets }
}

public class FacebookSearchManager : ISearchManager
{
    public IEnumerable<ResultItem> Search(string keyword){ //Search in the Facebook Api and returns a collection of Status }
}

public class ForumSearchManager : ISearchManager
{
    public IEnumerable<ResultItem> Search(string keyword){ //Search in another Api and returns a collection of Forum posts}
}

You controller could then receive an ISearchManager in his constructor

public SearchController:IController
{
    private readonly ISearchManager _searchManager;

    public SearchController(ISearchManager searchManager)
    {
	_searchManager = searchManager;
    }
}

And you can happily search in different store without changing anything in the controller… Well, not exactly, because your IoC controller doesn’t have a clue of what radio button is ticked when it creates your SearchController, therefore it cannot resolve the dependency on ISearchManager. What you need is a factory to create the correct instance of ISearchManager depending on a parameter.

public SearchController:IController
{
    private readonly ISearchManagerFactory _searchManagerFactory;

    public SearchController(ISearchManagerFactory searchManagerFactory)
    {
	_searchManagerFactory= searchManagerFactory;
    }

    public ActionResult Index(string provider, string keywords)
    {
	var searchManager = _searchManagerFactory.Create(provider);
	var results = searchManager.Search(keywords);
	…
    }
}

And the implementation of the factory often ends up like this :


public class SearchManagerFactory : ISearchManagerFactory
{
    public ISearchManager Create(string provider)
    {
	switch (provider)
	{
	    case “twitter” : return new TwitterSearchManager();
	    case “facebook”: return new FacebookSearchManager();
	    …
	}
     }
}

Well, that’s ok if you have a very small dependency graph, but you still, you start to spread the configuration of your dependencies in multiple places. You have an IoC container which decides which type each controller should receive, and you have this switch statement hidden inside a factory to decide which SearchManager. If my SearchManager have further dependency on, let’s say, a QueryBuilder (because the queries need to be built differently to satisfy the different APIs), it really becomes messy.

It would really be nicer if my factory could ask my container which implementation to return, with all its dependencies resolved. This turns out to be rather easy with most of the IoC containers.
  They allow you to assign a key to the different implementations of an interface. With StructureMap we have IContainer.GetInstance<T>(string key); Let’s see how we can use that in our factory.

public class SearchManagerFactory : ISearchManagerFactory
{
    readonly IContainerWrapper _container;

    public SearchManagerFactory(IContainerWrapper container)
    {
	_container = container;
    }

    public ISearchManager Create(string provider)
    {
	return _container.Resolve<ISearchManager>(provider);
    }
}

That was easy. Our factory is now asking our IoC container for a ISearchManager, but I don’t really want my factory to know which IoC Container I am using. Instead, I have this simple IContainerWrapper. Its implementation for StructureMap is straight forward :

public interface IContainerWrapper
{
    T Resolve<t>(string key);
}
public class StructureMapWrapper : IContainerWrapper
{
    private readonly StructureMap.IContainer _container;

public StructureMapWrapper(StructureMap.IContainer container)
{
    _container = container;
}

public T Resolve<T>(string key)
{
    return _container.GetInstance<T>(key.ToLower());
}


}

Now we’re all set. All we need to do is wire up all those components in the IoC configuration:

ObjectFactory.Initialize(x =&gt;
{
     x.For<ISearchManager>().Use<twittersearchmanager>().Named(“twitter”);
     x.For<ISearchManager>().Add<facebooksearchmanager>().Named(“facebook”);
});

ObjectFactory.Configure(x =>
{
     x.For<ISearchManagerFactory>()
       .Use(new SearchManagerFactory(new StructureMapWrapper(ObjectFactory.Container)))
);

This is the trick. I first set up the ISearchManager implementation, and give them a name in the Initialize method of StructureMap. Since I need to pass a reference of the container to my factory, the container needs to be already initialized when that happens. That’s why I set up the ISearchManagerFactory dependency after the initialization is done, in the Configure method.

That’s it, the dependency to the SearchManagers is now resolved by the IoC Container, depending on a runtime parameter. Why I think this is better? because that way, you keep all the configuration in one single place. You could even potentially change that configuration without recompiling if you configure your IoC using an XML config file.
