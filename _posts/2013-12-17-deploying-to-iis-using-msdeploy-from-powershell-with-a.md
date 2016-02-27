---
layout: post
title: Deploying to IIS using MSDeploy from powershell with a non-admin user
date: '2013-12-17T12:58:00+01:00'
tags:
- powershell
- CI
- msdeploy
tumblr_url: http://alsagile.com/post/70283855921/deploying-to-iis-using-msdeploy-from-powershell-with-a
---

In theory, msdeploy is perfect to deploy to IIS, but it was written for cmd, not for powershell. That means that when you call it from a powershell script, things start to fall appart…

I was pretty sure that I would find people who were doing the same thing on github, and indeed, It didn’t take long to find a trusted source: the NugetGallery website deployment script uses msdeploy

It was helpful, but that was not it… Ready to do the things the right way, I did not want to use an administrator account in my script as MSDeploy seems to want it.

And apparently there are ways to use a standard user with just the necessary rights.

The user you create on the machine where you want to deploy, needs to have the ACL rights to the folder of course, and he also needs to have the IIS Manager permission on the application that you want to deploy.
You then deploy to the Web Site url (for example, “Default Web Site”)
but you give an extra parameter to msdeploy that points to the Web Applilcation this time.

Summarized, it’s something like this :

{% highlight powershell %}
$AppName = "MyApp"
$Site = "Default Web Site"
$WebApp= "$Site/$AppName"

# Here we use the $Site variable, **Not the $WebApp**
$PublishUrl = "https://WebServer:8172/MSDeploy.axd?site=$($Site)"

$arguments = [string[]]@(
    "-verb:sync",
    "-source:package=''$Package''",
    "-dest:auto,computerName=''$PublishUrl'',userName=''$UserName'',password=''$Password'',authtype=''Basic''",
# This is where you point to the WebApp
    "-setParam:name=''IIS Web Application Name'',value=''$($WebApp)''",
    "-allowUntrusted")

# And this is how you call msdeploy with spaces in the arguments without problems
Start-Process $msdeploy -ArgumentList $arguments -NoNewWindow -Wait


{% endhighlight %}

But somehow that would not work, and the error message was not really helpful. It turns out, as most of the time, that I was doing it wrong (but in that case MSDeploy is really making it easy to shoot yourself in the foot)

I published the complete powershell wrapper that I use as a gist on github, I hope this helps someone. :)

https://gist.github.com/serbrech/8003858
