---
layout: post
identifier: cf6198ce67db41ba927bccb7c5463b53
title: Designing software for multiple platforms is not about making them similar
date: '2012-06-08T19:33:36+02:00'
tags:
- design
- github
tumblr_url: http://alsagile.com/post/24688024540/designing-software-for-multiple-platforms-is-not-about
---
Github builds great software. They think it through, design it well, and they ship good software.

Recently they shared their experience about how they designed the native windows github app.



What I find very interesting here is the principal advantage they see in doing this.
The first idea you might have when attacking this problem is to make the UI look relatively similar so that your product keeps it’s “identity” in your users eyes. But here is what they say :


  "We specifically made the decision to write each application in a language native to the platform, and this has turned out to be hugely beneficial to us. Because of this separation we’ve been free to tackle the problems that are most pressing for each platform and work in the best possible tools instead of being constrained to the lowest common denominator."


Instead of making the difference of the platforms a problem, they turned it into an advantage. They don’t think about how to make them similar despite the platform differences, on the contrary. Building the apps differently the different platforms gives a better experience, more adapted for every users. I believe this is the right approach.

Trying to make an app look and work the same on 2 different platforms might seem easier, or less complex from both the development side and the user’s side. But that would be ignoring the differences of experiences that are expected on the different on mac and windows, and it can drag you down towards the lowest common denominator, particularly in terms of user experience.

The last note that I really appreciate is the very last paragraph and sentence. I don’t think it needs much comments :


  "Design is often most important for the things that you don’t see."
