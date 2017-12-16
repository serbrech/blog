---
layout: post
identifier: c4619d152c7642e590a909574119120d
title: Testing the IDataReader mapping
date: '2012-09-13T14:24:00+02:00'
tags:
tumblr_url: http://alsagile.com/post/31458348401/testing-the-idatareader-mapping
---
I don’t want to enter the discussion if you should or not use a datareader, this is another topic. I will just show how I test the mapping between the reader and my objects easily.

First of all, I try to isolate the mapping from the rest of the code

I usually create one class by mapping and pass it the datareader. I use this base class :

{% highlight csharp %}
public abstract class DataReaderMapperBase<T> where T:class
{
      public abstract T Map(IDataReader reader);
}
{% endhighlight %}

A simple example of using this base class :

{% highlight csharp %}
public class PersonDataReaderMapper : DataReaderMapperBase<Person>
{
     public override Person Map(IDataReader reader)
     {
	   return new Person
	   {
		  FirstName = reader.GetString("Name"),
		  LastName = reader.GetString("LastName"),
		  ...
	   }
     }
}
{% endhighlight %}


But how to test this? there is a simple way of stubbing a datareader using a DataTable and a DataTableReader.
But that is still a cumbersome syntax, so I wrapped it in a helper that could take objects and transform them into a DataReader row. I will probably package a little better, but here is the raw draft, with an example of how you use it :

{% gist 3713904 %}
