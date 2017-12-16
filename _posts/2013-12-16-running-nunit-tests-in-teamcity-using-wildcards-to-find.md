---
layout: post
identifier: df27e02958234b61af18d21e77ac159f
title: Running Nunit tests in Teamcity using wildcards to find dlls
date: '2013-12-16T15:19:00+01:00'
tags:
tumblr_url: http://alsagile.com/post/70189963777/running-nunit-tests-in-teamcity-using-wildcards-to-find
---
Quick note for my future self to avoid an hunting for the imaginary problem.

When using wildcards to run nunit tests in Teamcity, use this :

    **\bin\debug\*Test*.dll


And not this :

    **\*Test*.dll


The reason being that if we don’t specify the bin folder in the path, it will pick up the dll in the obj folder, and that folder will not contain the referenced dlls, hence it will run the tests twice and fail on the one from the obj folder.
