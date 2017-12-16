---
layout: post
identifier: 5afbb093d5fe436c850dd07e68ac93aa
title: Calling roundhouse from powershell
date: '2014-01-09T14:04:26+01:00'
tags:
- powershell
- roundhouse
- CI
- teamcity
tumblr_url: http://alsagile.com/post/72762547008/calling-roundhouse-from-powershell
---
It’s not the first time I struggle to pass parameters containing spaces to an executable from powershell. And since I finally managed to get it right, I will not miss that chance to document it for my future self.

Pay attention to the back-ticks to escape the double quotes…

And pay attention to how it checks for the exit code (don’t forget the -PassThru parameter to get the process object out).

Here is the small powershell wrapper I used for the roundhouse excutable.

{% gist 8333695 %}
