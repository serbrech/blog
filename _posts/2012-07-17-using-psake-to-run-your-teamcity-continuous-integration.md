---
layout: post
identifier: 85783c8a48ec4b328c65bb617a2b24a5
title: Using psake to run your TeamCity continuous integration build
date: '2012-07-17T14:09:00+02:00'
tags:
- psake
- CI
- TeamCity
tumblr_url: http://alsagile.com/post/27403222324/using-psake-to-run-your-teamcity-continuous-integration
---
Recently I had to setup a CI build in TeamCity.
I already did this once using msbuild script, and met very quickly walls when I wanted to get into more advanced deployment scripts. So I looked for a different solution and found psake.
From the github project :


  psake is a build automation tool written in PowerShell. It avoids the angle-bracket tax associated with executable XML by leveraging the PowerShell syntax in your build scripts. psake has a syntax inspired by rake (aka make in Ruby) and bake (aka make in Boo), but is easier to script because it leverages your existent command-line knowledge.

DISCLAIMER : ![Works on my machine badge](http://25.media.tumblr.com/tumblr_m31xya0nn01qg8w62o1_250.jpg)
I decline all responsibilities if that does not makes you happy :)


The psake build script

At the top of my default.ps1 I have

{% highlight powershell %}
properties {
    $home = $psake.build_script_dir + "/../.."
    $sln_file_server = "$home/Server/Server.sln"
    $sln_file_client = "$home/Client/Client.sln"
    $sln_file_common = "$home/Common/Common.sln"
    $configuration = "Debug"
    $framework = "4.0"
}
{% endhighlight %}

My build script is not at the root of the repository, so I adjust a $home variable myself.

And here are my tasks :
{% highlight powershell %}

    task CompileCommon {
      exec { msbuild "$sln_file_common" /p:Configuration=$configuration }
    }

    task CompileClient {
      exec { msbuild "$sln_file_client" /p:Configuration=$configuration }
    }

    task CompileServer {
      exec { msbuild "$sln_file_server" /p:Configuration=$configuration }
    }

    task All -depends Clean, CompileCommon, CompileClient, CompileServer {
    }

{% endhighlight %}

Awesome, but the integration with teamcity has some surprises (fortunately relatively simple to overcome).
In short, Powershell does not use the exit codes of the programs you launch from it.
If msbuild exits with the code 1 (for failure), the powershell process will still exit with the code 0 (success). Unfortunately, this is what teamcity is looking at to tell if the build step passed or failed.

I ran into this problem and had a hard time making the build fail when the compilation was failing.
Here is how I solved it, which is similar to what the wiki says:

Make TeamCity fail when the compilation fails

I fixed this in the teamcity build step setup. I have tried multiple things (running the psake.cmd didn’t do it for me), and I ended up with the following configuration (click to see a higher resolution) :


[http://25.media.tumblr.com/tumblr_m7axvySja21qg8w62o1_500.png](http://24.media.tumblr.com/tumblr_m7axvySja21qg8w62o1_1280.png)

Some tips

To reference the paths in my build script, psake has this nice little helper : $psake.build_script_dir
This variable will have the path to the script that you execute. Whether you execute it from another folder or right from the script folder, it will hold the correct value. That’s what I base the path to the .sln files in the properties section

    $home = $psake.build_script_dir + "/../.."


That’s also how I let psake know where my powershell modules are in the psake.config.ps1 :

{% highlight powershell %}
$root = $psake.build_script_dir
$config.modules=("$root\modules\*.psm1")
{% endhighlight %}

Another useful tip is that psake has a built-in function to configure the build environment (access to msbuild.exe of a specific framework for example)

if you digg into psake.psm1 source code, you will find this function :

{% highlight powershell %}
function Configure-BuildEnvironment {
$framework = $psake.context.peek().config.framework
    if ($framework.Length -ne 3 -and $framework.Length -ne 6) {
	throw ($msgs.error_invalid_framework -f $framework)
    }
$versionPart = $framework.Substring(0, 3)
$bitnessPart = $framework.Substring(3)
$versions = $null
switch ($versionPart) {
    ''1.0'' {
	$versions = @(''v1.0.3705'')
    }
    ''1.1'' {
	$versions = @(''v1.1.4322'')
    }
    ''2.0'' {
	$versions = @(''v2.0.50727'')
    }
    ''3.0'' {
	$versions = @(''v2.0.50727'')
    }
    ''3.5'' {
	$versions = @(''v3.5'', ''v2.0.50727'')
    }
    ''4.0'' {
	$versions = @(''v4.0.30319'')
    }
    default {
	throw ($msgs.error_unknown_framework -f $versionPart, $framework)
    }
}

$bitness = ''Framework''
if ($versionPart -ne ''1.0'' -and $versionPart -ne ''1.1'') {
    switch ($bitnessPart) {
	''x86'' {
	    $bitness = ''Framework''
	}
	''x64'' {
	    $bitness = ''Framework64''
	}
	{ [string]::IsNullOrEmpty($_) } {
	    $ptrSize = [System.IntPtr]::Size
	    switch ($ptrSize) {
		4 {
		    $bitness = ''Framework''
		}
		8 {
		    $bitness = ''Framework64''
		}
		default {
		    throw ($msgs.error_unknown_pointersize -f $ptrSize)
		}
	    }
	}
	default {
	    throw ($msgs.error_unknown_bitnesspart -f $bitnessPart, $framework)
	}
    }
}
$frameworkDirs = $versions | foreach { "$env:windir\Microsoft.NET\$bitness\$_\" }

$frameworkDirs | foreach { Assert (test-path $_ -pathType Container) ($msgs.error_no_framework_install_dir_found -f $_)}

$env:path = ($frameworkDirs -join ";") + ";$env:path"
# if any error occurs in a PS function then "stop" processing immediately
# this does not effect any external programs that return a non-zero exit code
$global:ErrorActionPreference = "Stop"
}
{% endhighlight %}

That function is called automatically by psake before the tasks are run. It looks at psake’s configuration hash to find the  $framework property that you can set at the top of the default.ps1 file.
You maybe noticed that I do the following :
    properties {
	…
	$framework = “4.0”
    }

I can then do exec { msbuild … }   in my tasks without caring where it is located on my build server.
The best is that I can override the properties from the teamcity command line by passing another $framework value as a parameter.

    ./psake.ps1 -properties @{"framework"="3.5"}


Simple and efficient :)
