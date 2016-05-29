---
layout: post
title: Move msmq storage location with powershell and Desired State Configuration
date: '2016-05-29T18:00:00+02:00'
tags: powershell, dsc, msmq
---

Lately I've been writing powershell scripts to automate the provisioning of parts of an environment in Azure.

One of the tasks was to move the MSMQ storage from the C: drive to a faster E: drive. I already knew that touching msmq files meant trouble, and some quick googling confirmed that.

I thought that maybe you could choose where to run MSMQ when you install it. but I was wrong. You first have to install the feature to its default location, and then only can you move it.

I did not find any complete information that just made it work for me, hence this article. I case I have to do this again some time in the future.

OK, so how do we go about moving it?
It turns out that with Powershell 4.0, Microsoft added a nice little helper : [Set-MsmqQueueManager]("https://technet.microsoft.com/en-us/library/dn391736.aspx")

This requires the msmq service to be running, but you can use it like this :

{% highlight powershell %}
PS C:\> Set-MsmqQueueManager -MsgStore E:\MSMQ\ -TransactionLogStore C:\MSMQ\ -LogMsgStore E:\MSMQ\
{% endhighlight %}

And that will move the few already existing msmq files to the given folder. Great! But of course, it could not be so easy...
After running this, the msmq service will not be running, and it will not start. No panic. The reason is that the user running MSMQ does not have the rights to its files anymore.

And that's where I save my future self from a day of pulling my hair off trying to figure out how this works.
The files were created by SYSTEM, so you cannot change the ACL on it.
you first have to take ownership :

{% highlight powershell %}
PS C:\> takeown /F 'E:\msmq' /R /D y
{% endhighlight %}

then reset the inheritance settings of the acl on the files :

{% highlight powershell %}
PS C:\> icacls 'E:\msmq' /reset /T /C
{% endhighlight %}

and finally, give the rights to Network Service :

{% highlight powershell %}
PS C:\> icacls 'E:\msmq' "/grant:r" "NT AUTHORITY\NetworkService:(OI)(CI)(M)" /T /C
{% endhighlight %}

A closer look at these commands. Msmq runs as Network Services. The account needs rights on the msmq and LQS sub folder. A few things happen here.

- `/T` means traverse, so all subitems get the same rights.
- `/C` is to continue the process if it there is an error.
- We first `reset` because apparently the inheritance of rights has been screwed up. Just giving the rights to the user was not enough.
- And finally we `grant`. `:r` is to ensure that we override existing permissions. Might be overkill, but hey, I was frustrated.

Anyway, if you are still confused, you'll go read the [icacls documentation](https://technet.microsoft.com/en-us/library/cc753525.aspx).

Note the exact name of the Network Service account as well. This also cost me some googling hours...
And to wrap up, a little DSC script resource, so that I never have to deal with this ever again. Maybe I'll refactor this into a proper resource some time :

{% gist 164eeaea3635bfccdcd4d95ec0cbd433 %}
