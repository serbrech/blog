---
layout: post
identifier: fe7f2e842ec24b9aa2dc9866953d182c
title: How I startup all my project services in multiple terminal tabs in one command
date: '2012-04-25T21:27:00+02:00'
tags:
- script rails
tumblr_url: http://alsagile.com/post/21795518778/how-i-startup-all-my-project-services-in-multiple
---
#### DISCLAIMER
    I am running snow leopard, and TotalTerminal (because the slide down is awesome)


It’s been some time now that I am playing and learning ruby, rails, and the tools around it to develop.
On a project I work on, we start having a bunch of services to start to be ready to work. If you want to really see what is going on, you want each of them in a separate tab.
And so the dance starts:

    Cmd + T
    cd <project>
    [insert command to start service]


And the services to start are:
-mongod
-redis-server
-foreman
-passenger
-fork
-autotest

I looked for a way to script this repetitve work, and found how to open a new tab in your terminal using shellscript.

I edited it appropriately to pass it a command and Voilà :

{% gist 2492064 %}

And it is now in my .bash_profile with a better alias.
If you have better way, or equivalent ones, please tell me, I am always happy to learn!
