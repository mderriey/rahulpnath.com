---
layout: post
title: "Setting up Build and Deploy Pipeline for a .NET Core Console Application"
comments: true
categories: 
- .Net Core
- Programming 
tags: 
date: 2018-01-03
completedDate: 2017-12-31 06:40:46 +1000
keywords: 
description: Automatically deploy and run a console application using TeamCity and Octopus Deploy.
primaryImage: net_core_teamcity.png
---

I was given a console application written in .NET Core 2.0 and asked to set up a continuous deployment pipeline using [TeamCity](https://www.jetbrains.com/teamcity/) and [Octopus Deploy](https://octopus.com/). I struggled a bit with some parts, so thought it's worth putting together a post on how I went about it. If you have a better or different way of doing things, please shout out in the comments below.

At the end of this post, we will have a console application that is automatically deployed to a server and running, anytime a change is pushed to the associated source control repository.

### Setting Up TeamCity

Create a [New Project](https://confluence.jetbrains.com/display/TCD10/Creating+and+Editing+Projects) and add a [new build configuration](https://confluence.jetbrains.com/display/TCD10/Creating+and+Editing+Build+Configurations) just like you would for any other project. Since the application is in .NET Core, install the [.NET CLI plugin](https://github.com/JetBrains/teamcity-dotnet-plugin) on the TeamCity server.

<img src="/images/net_core_teamcity_build_steps.png" alt="Build Steps to build .Net Core">

The first three build steps use the .NET CLI to [Restore](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-restore?tabs=netcore2x), [Build](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-build?tabs=netcore2x) and [Publish](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore2x) the application. Thee three steps restore the dependencies of the project, builds it and publishes all the relevant DLL's into the publish folder. 

The published application now needs to be packaged for deployment. In my case, deployments are managed using Octopus Deploy. For .NET projects, the preferred way of packaging for Octopus is using [Octopack](https://octopus.com/docs/packaging-applications/creating-packages/nuget-packages/using-octopack). However, OctoPack does not support .NET Core projects. The recommendation is to either use [dotnet pack](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-pack?tabs=netcore2x) or [Octo.exe pack](https://octopus.com/docs/packaging-applications/creating-packages/nuget-packages/using-octo.exe). Using the latter I have set up a Command Line build step to pack the contents of the published folder into a zip (.nupkg) file.

``` bash
octo pack --id ApplicationName --version %build.number% --basePath published-app 
```

The NuGet package is published to the NuGet server used by Octopus. Using the [Octopus Deploy: Create Release](https://octopus.com/docs/api-and-integration/teamcity) build step, a new release is triggered in Octopus Deploy.

### Setting Up Octopus Deploy

Create a [new project](https://octopus.com/docs/deployment-process/projects) in Octopus Deploy to manage deployments. Under the Process tab, I have two [steps](https://octopus.com/docs/deployment-process/steps) - one to deploy the Package and another to start the application.

<img src="/images/net_core_octopus_deploy_process.png" alt="Octopus Deploy Process Steps">

For the Deploy Package step I have enabled Custom Deployment Scripts and [JSON Configuration variables](https://octopus.com/docs/deploying-applications/deploying-asp.net-core-web-applications/json-configuration-variables-feature). Under the pre-deployment script, I stop any existing .NET applications. If multiple .NET applications are running on the box, select your application explicitly.

``` powershell
Stop-Process -Name dotnet -Force -ErrorAction SilentlyContinue
```
Once the package is deployed, the custom script starts up the application.

``` powershell
cd C:\DeploymentFolder
Start-Process dotnet .\ApplicationName.dll
```

With all that set, any time a change is pushed into the source control repository, TeamCity picks that up, build and triggers a deployment to the configured environments in Octopus Deploy. Hope this helps!