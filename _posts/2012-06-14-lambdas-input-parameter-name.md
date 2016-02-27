---
layout: post
title: Lambdas input parameter name
date: '2012-06-14T00:16:00+02:00'
tags:
- graph
- neo4j
- lambda
- syntax
tumblr_url: http://alsagile.com/post/25048787678/lambdas-input-parameter-name
---
I just saw an example of lanbda using “it” as the lambda input parameter name, and I found it quite nice. I never thought about this, nothing exceptional, just a nice touch in readability :

    collection.Where( it => it.Name == "MyRelationship" );

It stays concise, still keeps a meaning when you read it, and it stays short.

By the way, this lambda was from an example in the Neo4J Rest interface .net wrapper that popped up in the github "C# Most Watched This Week" rss feed.

Maybe something I will look into. Graph databases are still a mystery to me.
