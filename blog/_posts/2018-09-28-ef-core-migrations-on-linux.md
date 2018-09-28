---
layout: post
title: 'Entity Framework Core Migrations on Linux'
subtitle: "Run Entity Framework Core Database Migrations While Deploying to Linux"
date: 2018-09-28 12:00:00
author: 'Matt Millican'
header-img: 'img/blog/abstract-servers-internet.jpg'
permalink: blog/:slug
---

As you've probably heard, ASP.NET Core runs on Linux (and macOS)! And it's actually pretty easy to get started. However, 
running Entity Framework Core database migrations on Linux after publishing is not as easy.

During development [on Linux and macOS], running migrations is as simple as always:

```
$ dotnet ef database update
```

Note that to use the `dotnet ef` command, you must either have version 2.1+ of the .NET Core SDK installed, or the 
`Microsoft.EntityFrameworkCore.Tools` NuGet package installed in your project.

~That's because the EF tools are installed on your system and easily accessible. Where I ran into issues however, is while
trying to deploy my application using AWS Code Deploy. Like most CI/CD systems, this is intended to be fully automated. Here are the
high-level steps to deploy this application. I'll cover the details of each step in a later blog post (or hopefully series). ~

1. From your development machine or build server, publish your application and zip it up:
    ```
    $ dotnet publish -c Release
    ```

1. Since Code Deploy relies on the above artifact being in an S3 bucket, put the file in the appropriate S3 bucket.

1. Kick off the Code Deploy process. In my case, there's a hook in the `appspec.yml` file to execute the migrations.

For example purposes, we're going to assume my application was deployed to my Linux server at `/var/www/html/api` and this will 
be our working directory. 

If we run the same database update command, we get an error:

```
$ dotnet ef database update
No project was found. Change the current working directory or use the --project option.
```

Well that's interesting. I'd gladly specify a project file, but since the app has been published already, it doesn't exist 
in our artifact. So now what? Turns out we need to be a little more verbose with our command:

```
# Note: I broke this up into lines to make it easier to read and explain

$ dotnet exec \
    --depsfile MyApiProject.deps.json \
    --runtimeconfig MyApiProject.runtimeconfig.json \
    
    "/opt/ef-tools/ef.dll" database update \
    --assembly MyApiProject.dll \
    --startup-assembly MyApiProject.dll \
    --root-namespace MyApiProject
```

When you run your EF commands from your development machine with the `.csproj` file, it gathers all the dependencies 
it needs from the project. Because we don't have the `.csproj` file after publishing, we have to tell `dotnet` where to find
those dependencies. Let's take a look at these lines individually.

- `dotnet exec` is responsible for running the command. We need to do this to provide the EF tools with the additional information 
missing due to the lack of `.csproj`
- `--depsfile` describes the dependencies the application has
- `--runtimeconfig` describes the runtime configuration for the app, including the SDK version
- `"/opt/ef-tools/ef.dll" database update` is calling out to the EF tools to actually run the database update command
- `--assembly` and `--startup-assembly` are telling EF where to find the migrations and `DbContext`. Note that dependening on your
project structure, these may be the same or could vary.
- `--root-namespace` Is the root namespace (or the assembly name) of the application

> As a side note, if you want to see what `dotnet` is running when you run `dotnet ef database update` locally, you can add the `--verbose` flag.
If there are migrations, you'll have to scroll up and you'll see a `dotnet exec` command similar to the above.

I'm really hoping the EF / .NET teams make this easier in the future. I've opened an [issue here](https://github.com/aspnet/EntityFrameworkCore/issues/13339)
 which talks about this, so please "up-vote" it if you'd like to see it as well. Also, a special thanks to Ben Day for [his post and script](https://www.benday.com/2018/07/05/deploy-entity-framework-core-2-1-migrations-from-a-dll/) 
pointing me in the right direction.

> In the [issue](https://github.com/aspnet/EntityFrameworkCore/issues/13339) comment thread, the .NET CLI team 
does not recommend "trying this at home" [referring to running `dotnet exec`] - use caution with other use cases.