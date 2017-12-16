---
layout: post
identifier: e4d8749a286844ffb1c1806efb42fbc5
title: Running Solr as a windows service
date: '2012-06-15T13:23:00+02:00'
tags:
- solr
- github
tumblr_url: http://alsagile.com/post/25153448145/running-solr-as-a-windows-service
---
[Solr](http://lucene.apache.org/solr/) is a very powerful search engine built on top of Lucene. If you read this article you are probably familiar with it already, so I will skip the introductions.

One recurrent issue is that it is not the best fit to run it on a windows server. Or at least, it is not straight forward [for everyone](http://stackoverflow.com/questions/2531038/how-to-run-solr-on-a-windows-server-so-it-starts-up-automatically) [apparently](http://stackoverflow.com/questions/9532337/running-jettysolr-on-windows-server).

I stumbled upon a [small project](https://github.com/rupertbates/SolrWindowsService) (thanks rupertbates) on github that is making this a piece of cake. I updated it and thought that might help others :

[https://github.com/serbrech/SolrWindowsService]

All you need to do is update the few settings in config file to match your environment :

{% highlight xml %}
    <solrService>
    <add key="JavaExecutable" value="C:\Program Files (x86)\Java\jre6\bin\java.exe" />
    <add key="WorkingDirectory" value="C:\Solr\apache-solr-4.0\example" />
    <add key="Solr.Home" value="solr" />
    <add key="CommandLineArgs" value="-Djava.util.logging.config.file=logging.properties" />
    <add key="ShowConsole" value="false" />
    <add key="Port" value="8983" />
    <add key="InstanceName" value="MyProjectName" />
    <add key="ClientSettingsProvider.ServiceUri" value="" />
    </solrService>
{% endhighlight %}

After compilation, run Install.bat from the bin folder.

That’s it. Solr is installed as a service on your windows machine.

if you just want the executable, [here you go](http://dl.dropbox.com/u/2565028/SolrWindowsService.zip).
