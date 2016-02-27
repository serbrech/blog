---
layout: post
title: Trim your build on Silverlight projects!
date: '2012-03-22T15:25:00+01:00'
tags:
- silverlight
- continuous integration
tumblr_url: http://alsagile.com/post/19731792868/trim-your-build-on-silverlight-projects
---
This post expose the problem from the current project and technology I work with. However, the general lesson behind it can be applied to any other project or technology.

By curiosity, I checked the build time stats fromour Continuous Integration server (TeamCity) for our main Silverlight project. It’s big, and the compile time is a clearly pain on this platform.

Here is what I saw :



A clear drop happened. What did we do for the build time to drop by nearly 30 seconds?

Looking at the history, what happened is that we cleaned up the styles from our solution. We use the Telerik suite of controls, and they come with a massive style library. Over time, we accumulated lots of them in our projects. some where not even used anymore. or styles from controls that we don’t use where hanging around.

Yep. Parsing, compiling and packaging this file was adding 20 (TWENTY) seconds to the build time!

Before: 1min 57sec (not that a few days before it was even above 2 min)
After: 1min 37sec

Let’s do the math

I bet we would get under 1 min build time if we would clean up further the styles. (note that 1min build is still really long on any other technology than Silverlight…)

How many time/day do we build the client?
Let’s say 20 times/day/developer.
If we manage to trim 1min in total. (from the initial 2min to about 1 min), we would save 20min per day per developer. That’s 1h40/week/dev
Consider 3 developers working on that project, that’s 5 man hours per week lost waiting for compilation! 20 man hours per months on compilation! imagine if you have 20 developers working on that project!

And we don’t consider the extra time lost because the developers change focus, go get a coffee or check their mail while waiting for the compiler.
I’m also pretty sure that Visual Studio would be very happy to have less files/code/xaml to handle and would hang much less.

So how much is compile time worth working on? Why don’t you spend 5h getting that compile time down?
