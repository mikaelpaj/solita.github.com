---
layout: post
title: Installing server environment with PowerShell
author: Rinorragi
excerpt: Automize your IIS webserver installation with PowerShell 
categories: 
- Episerver
tags: 
- Episerver 
- DOTNET 
- PowerShell
---
Most likely, as a reader you are mostly interested in my scripts. Here are my scripts for setting up Windows Server 2012 R2 ready for WebDeployments and running EPiServer ASP.NET site!

* [Example script using modules below](https://github.com/solita/powershell-webdevelopertools/blob/master/scripts/solita_example_server_install.ps1)
* [Config used by example script](https://github.com/solita/powershell-webdevelopertools/blob/master/scripts/solita_example_server_install_config.xml)
* [IIS tools module](https://github.com/solita/powershell-webdevelopertools/blob/master/solita-iistools.psm1)
* [Server tools module](https://github.com/solita/powershell-webdevelopertools/blob/master/solita-servertools.psm1)

## Why automate the installation?

This blog is about how to make your brand new Windows Server ready for WebDeployments with just pressing enter once. Cloud services can make your infrastructure lifecycle handling very easy; still people often encounter situations where they need to host our ASP.NET applications in virtual machines or directly on physical hardware. On those situations installation procedures in Windows operating systems are often done with some "clickety click" magic that won't take too long. Still the "clickety click" installations have lots of long-term problems: 

* Installations can't be reproduced 
* Only the installer knows how he did it
* If you have multiple servers you are doing same manual steps multiple times
* Base of the installations does not differ much from project to project 
* After few years when your windows server needs upgrade you will need to repeat this
* There is no way to test "clickety click" installations 
* Windows Server Core installations are rare because people is not used to manage Windows Servers without GUI 

One could argue that documentation and clear processes would take care of all the problems above. Maybe they could but I have never seen installation documentation that has 100% coverage over how the installation has been done. Installation script works as document and developers are more likely to update it! 

## What needs to be installed on a fresh Windows server
Here is short version of my list what I would do for new Windows Server 

* Install IIS
* Install WebPI 
* Install newest .NET 
* Install webdeploy
* Install various modules for IIS from windows feature list or with WebPI 
* Install tools for administration (7zip, text editor etc) 
* Create IIS site and application pool and change some defaults for better

## Is there PackageManagement for Windows?
Oh yes there is! It is called [Windows PackageManagement](http://blogs.technet.com/b/packagemanagement/archive/2015/04/29/introducing-packagemanagement-in-windows-10.aspx) (aka OneGet). Unfortunately it is not available for Windows Servers yet. Production preview of Windows Management Framework 5 is already [available](https://www.microsoft.com/en-us/download/details.aspx?id=48729). Windows PackageManagement is actually a manager that can access multiple type of repositories to have unified way to download and install software from multiple sources. Microsoft MSI installers, Windows Features and software could be all installed with Windows PackageManagement in the future. The thing still missing is grown up economy where all the toys would be easily available in a secure manner. Chocolatey is pretty good already but it is missing a lot of software and I'm still sceptical about security of software packages. Also Chocolatey is not the main delivery channel for many software companies and there might be a huge delay before software is available from there. So instead of using Chocolatey I provide examples how to do silent installations with different type of software packages. 

## How stuff is installed without Chocolatey 
If you are familiar with Windows Server installation you might notice that there are multiple different types of installation involved in the process. Windows features, executables, msi files and software specific plugins are all in the same process. This is the part where I wish that Windows PackageManagement (aka OneGet) or Chocolatey can one day solve unifying installation. 

#### Windows features (e.g. IIS)
The first question you should be looking at is "what are the windows features"? To be able to install the features you must first learn how to find them. Here is a oneliner that will give the available features to you. 

```powershell
Get-WindowsOptionalFeature -Online 
```

Great! Now we know how to find the windows features. Now we just need to change the oneliner a bit to be able to Install them. This is a oneliner for .NET:  

```powershell
Enable-WindowsOptionalFeature -Online -All -FeatureName NetFx4
```

All-flag means that it will also install required dependencies if any. The .NET delivered among the operating system might not be latest, but the feature needs to be enabled all the same. IIS will need it among some other features. Full list of my users can be found in our example config file at the bottom of the article.

#### Executables 
With windows feature we were able to install the .NET version that is delivered with the operating system. Although that is unlikely the newest one. Instead we need to fetch the newest one from [the internet](https://download.microsoft.com/download/E/4/1/E4173890-A24A-4936-9FC9-AF930FE3FA40/NDP461-KB3102436-x86-x64-AllOS-ENU.exe). With executables you never can be quite sure how they should be installed because there is no strictly unified commandline parameters. Here is an example how your executable installation might look like. 

```powershell
$arguments = @(
        "/qn" # display no interface 
        "/norestart"
        "/passive"
		"/S")
Start-Process $exe  -Wait -PassThru -Verb runAs -ArgumentList $arguments
```

I chose an approach where I just guess what kind of parameters the executable will need for silent installation. I found out that /S is pretty common, but sometimes also /passive is used too. Most likely your executables needs to be run elevated, that can be achieved with "-Verb runAs" parameter for Start-Process function. So even if this worked for my executables, this might be terrible mess for yours (for example if there is checks for arguments being eligible). 

#### MSI files 
MSI files are really common way to install stuff in Microsoft environment. They are also unified so we can be more certain that silent installation can be achieved. The installation looks really similar to installation of executables. 

```powershell
$arguments = @(
        "/i" #install product
        "`"$msiFile`""
        "/qn" # display no interface 
        "/norestart"
        "/passive")
Start-Process msiexec.exe -Wait -PassThru -Verb runAs -ArgumentList $arguments
```

The File to be installed is given as an parameter to msiexec.exe among some other flags. Installer is again run as elevated process to make sure that it can make all the configuration changes it wants. 

#### WebPI modules
To be able to install WebPI modules, you need to first [install it](http://download.microsoft.com/download/C/F/F/CFF3A0B8-99D4-41A2-AE1A-496C08BEB904/WebPlatformInstaller_amd64_en-US.msi). After installation you are able to use it's cmdline tools to install modules. Before you know what to install, you might need to list the available modules. Here is an example oneliner for listing:

```powershell
& (Join-Path "$env:programfiles" "microsoft\Web Platform Installer\webpicmd.exe") /List /ListOption:All
```

Once we get all the available modules we can find out that the name of the needed WebDeploy package is WDeploy36NoSmo. Here we install WebDeploy with oneliner: 

```powershell
& (Join-Path "$env:programfiles" "microsoft\Web Platform Installer\webpicmd.exe") /Install /Products:"WDeploy36NoSmo" /AcceptEula
```

AcceptEula flag basicly gives you silent installation. You might want to elevate this process too, but I didn't feel that it was necessary. 

## How stuff is configured
Once we got our IIS, .NET, WebPI and WebDeploy successfully installed there are still few steps before machine is ready to host our applications. First we need to make sure that the IIS website is set to be able to make deployments. 

#### IIS website 
First of all we need a folder for IIS website. We can create it like this: 

```powershell
$folderPath = "C:\MySite"
if(!(Test-Path -Path $folderPath )){
	Write-Verbose "Creating installation folder $folderPath"
	$Null = New-Item -ItemType directory -Path $folderPath
}
```

Then we need to setup an application pool for the website. It is also good idea to make sure few defaults are changed to better meet the purpose. We like to disable application pool idle timeout to get rid off unexpected expensive first hits. Instead we are using periodic restart to restart the pool when we are not expecting many customers.  

```powershell
Import-Module WebAdministration
New-WebAppPool -Name $appPoolName
# .NET runtime 
Set-ItemProperty "IIS:\AppPools\$appPoolName" -Name "managedRuntimeVersion" `
	-Value $appPoolDotNetVersion
# recycling
Set-ItemProperty "IIS:\AppPools\$appPoolName" -Name Recycling.periodicRestart.time `
	-value ([TimeSpan]::FromMinutes(1440)) # 1 day (default: 1740)
# Clear existing values if any
Clear-ItemProperty "IIS:\AppPools\$appPoolName" -Name Recycling.periodicRestart.schedule 
Set-ItemProperty "IIS:\AppPools\$appPoolName" -Name Recycling.periodicRestart.schedule `
	-Value @{value="$appPoolRestartTime"}
Set-ItemProperty "IIS:\AppPools\$appPoolName" -Name processModel.idleTimeout `
	-value ([TimeSpan]::FromMinutes(0)) # Disabled (default: 20)
# logs from recycling to event log
$recycleLogEvents = "Time,Requests,Schedule,Memory,IsapiUnhealthy,OnDemand,ConfigChange,PrivateMemory"
Set-ItemProperty "IIS:\AppPools\$appPoolName" -Name Recycling.logEventOnRecycle `
	-Value $recycleLogEvents
```

After that we can create the website. Below there is an example how to create website with possible multiple bindings:

```powershell
New-WebSite -Name $siteName  `
	-Port 80 `
	-HostHeader $siteName `
	-PhysicalPath $physicalPath  
Set-ItemProperty IIS:\Sites\$siteName -name applicationPool `
	-value $appPoolName
# set bindings (go through all the bindings and create new webbinding for each)
foreach($binding in $bindings)
{
	Write-Verbose "Adding binding $binding"
	$bindingProtocol = "http"
	$bindingIP = "*"
	$bindingPort = "80"
	$bindingHostHeader = $binding.Hostname
	$bindingCreationInfo = New-WebBinding -Protocol $binding.protocol `
		-Name $siteName `
		-IPAddress $bindingIP `
		-Port $bindingPort `
		-HostHeader $bindingHostHeader
}
```

#### WebDeploy
Once our website is happily set we can easily configure it for WebDeploy with the WebPI plugin we installed earlier. Scripts for WebDeploy can be found under the installation directory of the product. The scripts take care of problems like giving permissions to the iis site folder for webdeploy, creating user if it does not exist and setting up the WebDeploy configurations. Here is how to use the script with one long oneliner: 

```powershell
& (Join-Path "$env:programfiles" "IIS\Microsoft Web Deploy V3\scripts\SetupSiteForPublish.ps1") `
	-siteName $siteName `
	-siteAppPoolName $appPoolName `
	-deploymentUserName $wdeployUser `
	-deploymentUserPassword $wdeployUserPw `
	-managedRuntimeVersion $appPoolDotNetVersion
```

## Summing up 
Instead of huge scripts I like to put my PowerShell stuff to modules and also I like to be able to configure my installations. Thus I created few modules, an example script and an example configuration. I am using XML configuration because it was the way to go earlier with the PowerShell (it was easy to load unlike JSON). Now newer PowerShell has "ConvertFrom-JSON" function that makes it possible to use also JSON configuration. 

Here is how to load an XML file: 

```powershell
[xml]$config = get-content $configFile 
```

After the variable with XML file contents is loaded it is easy to access different elements inside it. For example if you would like to access the name of the WebDeploy user it might look something like this:

```powershell
 $config.Root.IIS.WebDeploy.GetAttribute("Name")
```

With all this done you could also enable PowerShell remoting on your server and do all the installations without using any RDP connection at all. 