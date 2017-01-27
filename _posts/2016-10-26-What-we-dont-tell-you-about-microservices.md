---
layout: post
title: What we don't tell you about microservices
excerpt: My current client has a long lived software that has evolved over about 10 years. It is a large codebase, that started as monolithic. The system was slowly split into services and of course, WCF was used to make them communicate.
date: '2016-10-27T08:15:00+01:00'
tags: microservices, architecture, devops
image: "/blog/assets/article_images/2016-10-26-what-we-dont-tell-you/Menhirs_carnac.jpg"
image2: "/blog/assets/article_images/2016-10-26-what-we-dont-tell-you/Menhirs_carnac.jpg"
---

# Setting the stage

My current client has a long lived software that has evolved over about 10 years. It is a large codebase, that started as monolithic.
The system was slowly split into services and of course, WCF was used to make them communicate. 
Trying to follow good SOA design, Message Queues and NServiceBus were introduced before it was even cool.
This multi-services system, although "distributed", is really just one big monolithic system,
with strong inter-dependencies, running in multiple processes.  
The single big bang deployment of the whole code base once a month is a good sign of that.   

For many (good and bad) reasons, writing small(er) services that can be deployed independently from the monolith has been the direction it took.
The first ones were adventurous, and paved the way for more teams to try it, which paved the way further...  
And here we are today, with about 10 teams of developers all writing new small services, web apis, web components, spas, etc...

We are doing microservices!  

<iframe src="//giphy.com/embed/yidUzHnBk32Um9aMMw?html5=true&playOnHover=true&hideSocial=true" width="480" height="240" frameborder="0" class="giphy-embed" allowfullscreen=""></iframe>

# Hang on a second... it's not that easy.

I work a lot on the devops side of the system, and here are some of the things that people don't tell about microservices.  

Most teams now understand how to build an autonomous isolated service that integrate with The Monolith. But we notice some difficulties; 
Some recurrent frustrations.  
I’ll just write down some facts that I think are important to consider in our environment.

* Every team develop new services nearly every release, if not every week.
* We probably doubled the number of services in the past year. (SPAs, Web apis, wcf endpoints, windows services, infrastructure services)
* Only a few developers are involved in the deployment that occurs once a month.
* It becomes impossible for 1 person to know if the system is 100% OK. (let that sink in)

This was to be expected. The teams are getting better at writing small focused services, but the fact is that the system is not ready for the consequences.  

**It is a DevOps nightmare.**  

The bottleneck and the friction for everyone is not development, **it’s everything else**. Cross cutting concerns, platform, devops.  
Some questions that every team has, for each new service :   

* How do we host this service, and where? 
* Should I host in the cloud? What platform should I use? How to deploy, monitor, talk to The Monolith, …?
* What about configuration management? (that one is a biggy)
* IIS Setup?
* Authentication?
* Logging?
* What storage can we use? SqlServer, Ravendb, DocumentDB? Something else?
* How do I version my stuff?
* What about test environments?
* How do I expose my service publicly?
* What about security?  
* Etc,...

Compare this to the workflow from about a year ago :  

* Adding code to an existing module.
* Merge.
* Go get coffee while the world is being compiled and tested by the CI builds.  

Microservices is not making things simpler, or easier. It gives you flexibility, speed of development, and more freedom, YES.  
**But**  
It increases *tremendously* the complexity of the overall system (architecture and infrastructure). And that’s where the friction comes from.  

There are ways to survive this, to support microservices and make them really shine. I'll probably blog about it too. 
