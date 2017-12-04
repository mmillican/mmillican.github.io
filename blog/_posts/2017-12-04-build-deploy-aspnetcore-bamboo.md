---
layout: post
title: 'Build ASP.NET Core Applications with Bamboo'
subtitle: 'Build and deploy ASP.NET Core web applications to IIS servers using Bamboo'
date: 2017-12-04 12:00:00
author: 'Matt Millican'
permalink: blog/build-deploy-aspnetcore-bamboo
disqus_identifer: build-deploy-aspnetcore-bamboo
disqus_url: /blog/post/build-deploy-aspnetcore-bamboo
redirect_from: /blog/post/build-deploy-aspnetcore-bamboo
---

For the last several years, my team at work has been using [Atlassian's Bamboo](https://www.atlassian.com/software/bamboo) build server to build and deploy our ASP.NET Web Applications.  I had a good process down for those projects, but with [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) came a whole new challenge.  After a bit of tinkering and trial and errors (not to mention, several failed builds), I finally got our .NET Core (v 2.0 as of this writing) to build and deploy via Bamboo.   Follow along to see the tasks involved.

> Note: I'm sure there are better ways to compartmentalize these tasks (and make them re-usable) which I am currently experimenting with.  I hope to do a follow-up post on this.

## Install ASP.NET Core SDK

Much like your local development machine, in order to build ASP.NET Core applications on your build server, you'll need to install the SDK. The easiest way to find this is via Microsoft's [dot.net downloads](https://www.microsoft.com/net/download/core) page.  Make sure the "SDK" 
tab is selected and choose the version appropriate for your server (or machine).

Once installed, run the following command to ensure it's installed and you have the latest version (2.0 as of this post):

```
> dotnet --version
```

At this point, your build server can build ASP.NET Core applications just as your development machine can.  If you have a project's source code already on the build server, I'd recommend building it via the command line to make sure it works.  It'll save you time and agony later.

## Server/IIS Setup

For this post, I'll be using IIS 7+ and the Web Management Service to do the deploys.  I'm not going to go into the specifics of installing and configuring the service, but here are a few things to ensure at the server level.  If you want to use a Window's user, make sure the user is created and you have the appropriate credentials.  I haven't had the greatest luck with IIS users, but that is also an option.  In IIS, with the server node selected, double-click "Management Service" to verify or edit its properties.  Verify that remote management is allowed and that "identity credentials" is set to the correct value for your environment.

On a site level, you'll have to grant your "deploy" user permissions to deploy the site.  This entails two main items (again, depending on your environment).   The first is to grant access to the Windows directory where your application will reside.  I've found that modify generally works well, but you'll have to allow the user to be able to delete folders.  You'll also have to grant the user IIS Manager permissions.  To do this, in IIS, select the website or application you're deploying to, and open "IIS Manager Permissions" at the bottom.  In the right column, click "Allow User" and enter or select the appropriate username and then click "OK". 

## Bamboo Tasks

To keep this a little shorter, I'm going to assume you have already created the basic build plan in Bamboo and have it connected to your repository.  When you create a new plan, Bamboo automatically creates a single stage.  For simplicity, we're going to stick with the single stage, but I'd recommend using multiple stages for things like database migrations vs deploying the application.  When you click on the stage on the left side, you should see the screen where you manage your tasks.  Let's get into the tasks.

1. **Source Code Checkout** 
    By default, Bamboo created the "Source Code Checkout" task when the plan (and stage) were created.  It should already be tied to your repository for the plan, but double check to make sure.  Next up is the fun part; creating tasks.  As of right now, I'm using "script" tasks for all the tasks, but I hope to change that soon.

1. **Restore Packages** 
    The first step you'll need to do, just as Visual Studio would, is to restore the Nuget packages for your project.  To do this, click "Add task" at the bottom of the task list.  A modal window will open with a variety of options.  In this case you want a "script" task.  Once selected, the modal will disappear and you'll get a new task form on the right side.  To start, give your task a name; mine is "Restore Packages".  Select "/bin/sh or cmd.exe" in the "Interpreter" drop down and make sure "Inline" is selected for "Source".  In the "Script Body", enter `dotnet restore` (this is where the magic starts).  Also make sure that the "Working sub directory" is set to the proper directory for the `.csproj` file.  I'd recommend creating plan variables for this.  If you do, you can enter something like `${bamboo.proj.project_dir}` for the sub directory.

1. **Clear Previous Publish**
    Because there is currently no simple way to use `msdeploy.exe` with `dotnet publish` in one fell swoop, I had to do a "temporary" local build + publish.  Because I wanted to be sure each build starts with a "clean slate," I have a task to clear out the previous build.  I found this easiest to do in Powershell, so following the same steps as "Restore Packages" above, create a new task.  The one change is that the interpreter will be "Windows Powershell".

   ```powershell
   
   $BuildTempDir = "${bamboo.build.working.directory}\${bamboo.proj.build_temp}\"
   Write-Host $BuildTempDir
   
   $WebPublishDir = $BuildTempDir + "_web\"
   If (Test-Path $WebPublishDir)
   {
       Remove-Item $WebPublishDir -Recurse
    }
    
    $DeployZip = $BuildTempDir + "_deploy.zip"
    If (Test-Path $DeployZip)
    {
        Remove-Item $DeployZip
    }

    ```

1. **Build + Temp Publish**
    Building the project is essentially the same as building via the CLI on your dev machine.  Similar to the previous steps, we will create a new script task with a "cmd.exe" interpreter.

    ```
    dotnet publish ${bamboo.proj.project_dir} --configuration ${bamboo.build_config} --output "${bamboo.build.working.directory}\\${bamboo.proj.build_temp}\web"
    ```

    The script is doing the standard `dotnet publish` for a "Release" configuration.  The important part here is that I am specifying a custom output path via the `--output` switch, so that I can use that artifact for the deploy later on.

1. **Database Migrations (optional)**
    If your project has a database and you're using Entity Framework Core Migrations, you'll likely want to deploy those migrations within your deploy as well.  This is also pretty straightforward, with a few exceptions.

    Just like the Build task above, create a script task with a "cmd.exe" interpreter.  In the "Script body", you will want to put the following (substitute for your `DbContext` names):

    ```
    dotnet ef database update --context ApplicationDbContext
    ```

    If you only have a single context, you can omit the `--context` switch.  Similarly, if you have multiple contexts, you can simply add additional lines to the script.  

    For this step, we need to define a few more items.  The first is "Environment variables".  Assuming you're deploying to a production environment, you'll want to set it to `ASPNETCORE_Environment=Production`.  If you are deploying to something other than production, you can change the value as appropriate.  Note, that similarly to when you are running the [web] app, EF Migrations will try to use the connection string in `appsettings.Production.json` if it can find one.  

    I also set my "Working sub directory" to `${bamboo.proj.project_dir}`; but you could probably specify the paths in the script above instead.

1. **Deploy**
    Finally, the home stretch.  Let's get this app deployed to your web server.  For purposes of this post, we're going to use MS Deploy and the Web Management Service.  This setup has worked relatively well for my team (for ASP.NET and ASP.NET Core projects), though it'd be nice to have something a little different for .NET Core.

    Again, similar to previous steps, we'll create a script task and set the interpreter to "cmd.exe".  Because there are a few things going on here, I'll share the script and explain it after.

    ```
    "C:\Program Files (x86)\IIS\Microsoft Web Deploy V3\msdeploy.exe" -verb:sync ^
        -source:iisApp="${bamboo.build.working.directory}\${bamboo.proj.build_temp}\web" ^
        -allowUntrusted:true ^
        -dest:iisApp='${bamboo.iis_site_name}',ComputerName="${bamboo.env.prod.msdeploy.service_url}?site=${bamboo.iis_site_name}",UserName='${bamboo.env.prod.msdeploy.user}',Password='${bamboo.env.prod.msdeploy.password}',AuthType='Basic',skipAppCreation=true ^
        -enableRule:AppOffline
    ```

    Using the "msdeploy" executable `sync` verb, we're essentially telling MSDeploy to sync two IIS applications.  The first in this case is just a Windows directory, but the principle still applies.  The source is defined by `-source:iisApp="--your output directory from step 4--"` and the destination is your IIS site/application's name.  Note all of the properties defined for the destination.  Most of them are self-explanatory, but pay attention to these:

      - `-allowUntrusted:true` = Interally, if the server does not have a certificate, you'll need this
      - `skipAppCreation=true` = I set up our applications before hand, so this ensures MSDeploy does not try to re-create it
      - `-enableRule:AppOffline` = This is probably the most important part of this step.  Without this, IIS will not stop the Kestrel process, and you won't be able to overwrite your application's DLL file(s).


And that's it!  If you run your job, your application should be deployed momentarily.  There are several ways to break these tasks up and to compartmentalize them more, which I hope to go into a future post.  However, I think this provides a straightforward way to get started, especially if you are unfamiliar with Bamboo and/or ASP.NET Core builds.  I'm sure similar concepts could be applied to other CI build systems such as Jenkins and Team City.  If you have any nifty tricks or trips, please share them in the comments!