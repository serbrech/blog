---
layout: post
title: Move msmq storage location in powershell with Desired State Configuration
date: '2016-05-25T23:00:00+02:00'
tags: powershell, dsc, msmq
---
# Move msmq storage location in powershell with Desired State Configuration

Lately I've been writing powershell script to automate the provisionning of parts of an environment in Azure.

One of the tasks was to move the MSMQ storage from the C: drive to a faster E: drive. I already knew that touching msmq files was usually trouble, and some quick googling confirmed that.

I expected however that you could choose where to run MSMQ at installation time. but I was wrong. You first have to install the windows feature to its default location, and then only move it.

OK, so how do we go about moving it?
It turns out that with Powershell 4.0, Microsoft added a nice little helper : [Set-MsmqQueueManager]("https://technet.microsoft.com/en-us/library/dn391736.aspx")

This requires the msmq service to be running, but you can use it like this :

    PS C:\> Set-MsmqQueueManager -MsgStore E:\MSMQ\ -TransactionLogStore C:\MSMQ\ -LogMsgStore E:\MSMQ\

And that will move the few already existing msmq files to the given folder. Great! But of course, they could not make it so easy...
After running this, the msmq service will be stopped, and will not start. The reason is that the user running MSMQ does not have the rights to its files anymore.

And that's where I save my future self of a day of pulling my hair off trying to figure out how this works.
The files were created by SYSTEM, so you cannot change the ACL on it.
you first have to take ownership :

    takeown /f E:\msmq /r

then reset the inheritance settings of the acl on the files :

    paste command here

and finally, give the rights to Network Service :

    bla bla NT Authority\NetworkService

Now all this together in a little DSC resource so that you never have to deal with this ever again :


    code for the resource
