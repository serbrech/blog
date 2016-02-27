---
layout: post
title: Find the local path from a local UNC network share path with Powershell
date: '2013-12-18T09:38:14+01:00'
tags:
- powershell
tumblr_url: http://alsagile.com/post/70376571644/find-the-local-path-from-a-local-unc-network-share-path
---
Just making sure that I will know where to find this if I need it again in the future.

If you have a UNC network share path, such as \Computer1\Shared and you need to know the local folder linked to that when running a script from Computer1,  you can achieve that easily in powershell :

    gwmi win32_share | ? {$_.Name -eq "Shared" } | select -expand path
