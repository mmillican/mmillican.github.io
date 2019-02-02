---
layout: post
title: 'Scaffolding ASP.NET Core API Controllers'
subtitle: "Quickly create API controllers to jump start your project"
description: "Quickly create API controllers to help jump start your next ASP.NET Core Web API"
date: 2018-03-27 12:00:00
author: 'Matt Millican'
header-img: 'img/blog/scaffolding.jpg'
permalink: blog/aspnetcore-controller-scaffolding
---

Starting a new project with many entities (objects) that require API endpoints can be quite daunting.  I recently faced this issue when I wanted to start up a new hobby project, based on a program called [JMRI](http://jmri.org) (Java Model Railroad Interface).  Though I am only looking at very small subset of this program, there are still roughly 20-30 entities to create.  Having to create the entities _and_ create controllers for each of those was enough to start convincing me to not tackle this project.

> For those unfamiliar, one of my hobbies is model railroading.  JMRI is a program used for setting up realistic, prototypical operations.

This project was to be developed in [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) with a Web API backend and [Vue.js](https://vuejs.org/) front-end.  I also wanted to develop this project completely on my Mac to ensure it was cross-platform (as it will eventually run on a Raspberry Pi), and as a learning experience.

So, after creating all of my entities (or a subset of them for testing), I set out to look for a CLI alternative to creating new API controllers in Visual Studio.  Enter `aspnet-codegenerator`.

## Setting up the generator

The `aspnet-codegenerator` CLI tools require a Nuget package, which is not included by default when creating projects via `dotnet new`.  To add this package, open your project's `.csproj` file and add the following:

```xml
  <ItemGroup>
    <DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="2.1.0-preview1-final" />
  </ItemGroup>
```

> Note, I'm using the 2.1 preview packages for this project.

After installing the Nuget package, you can run `dotnet aspnet-codegenerator` to see what generators are available.  Near the bottom of the output, you should see something like:

```
Available generators:
  view      : Generates a view.
  area      : Generates an MVC Area.
  controller: Generates a controller.
  identity  : Generates an MVC Area with controllers and
  razorpage : Generates RazorPage(s).
```

## Setting up the project

In order for the generator to have something to generate, we'll need to create a `DbContext` with at least one entity.  Consider the following _simple_ entity for our example:

```c#
public class Road 
{
    public int Id { get; set; }

    public string Name { get; set; }
}
```

We'll also need a simple DbContext to go along with this, so we'll use the following:

```c#
public class RailOpsContext : DbContext
{
    public DbSet<Road> Roads { get; set; }
    
    public RailOpsContext(DbContextOptions<RailOpsContext> options)
        : base(options)
    {    
    }

    // Remainder of the context class has been redacted for brevity
}
```

## Generating our first controller

Now that we have have our entity, DbContext (and appropriate configurations) set up, we're ready to get going with the scaffolding.  I'm going to share the full command and then explain each part and its significance 

> (Note: the `\` at the end of each line are for line breaks in the CLI.  They are used here for clarity but not required for running the commands).

```
dotnet aspnet-codegenerator controller \
    -name CarsController \
    -api \
    -async \
    -m RailOps.Api.Entities.Roster.Car \
    -dc RailOpsContext \
    -namespace RailOps.Api.Controllers \
    -outDir Controllers
```

| Fragment | Description |
| --------- | ---------- |
| `dotnet aspnet-codegenerator controller` | Invoke the command, specifying that we want to use the "controller" generator |
| `-name RoadsController` | The name parameter will be used for both the class name and the filename (`RoadsController.cs`) |
| `-api` | We want to create an API controller (could also be an MVC controller if you wish) |
| `-async` | Indicate that we want our actions / methods to be async methods |
| `-m RailOps.Api.Entities.Roster.Road` | Specify the fully qualified name of the entity for which we want to create a controller |
| `-dc RailOpsContext` | The name of the DbContext the controller should use |
| `-namespace RailOps.Api.Controllers` | The namespace in which you want to create the controller (if not specified, it will create the class in your root namespace) |
| `-outDir Controllers` | The directory (in the file system) where your newly generated file should be output (if not specified, the file will be created in the root of the project) |

Ater running this command, you should have a fully functional `RoadsController` that you can make HTTP requests against.  Inspecting the controller, you'll notice that proper HTTP verbs for GET, POST, PUT and DELETE have been created.

## Drawbacks

Now, there are a few "issues" I have with this approach:

- Typically, you'd want some sort of API model being exposed via your API rather than the entity representing the database.  These may have no navigational properties, or redact "private" properties you don't want exposed
- There is no validation.  For many obvious reasons, all HTTP requests should be validated

As my personal project matures, I will update my controllers to consider both of these reasons, but this will allow me to focus on core functionality of my application to start.  Hopefully the ideas presented in this post will help you get started more quickly on your next quick API project.
