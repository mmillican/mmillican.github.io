---
layout: post
title: 'Shared View Components with Different Presentations'
subtitle: "Use multiple views to render a view component differently"
date: 2018-01-31 12:00:00
author: 'Matt Millican'
permalink: blog/shared-view-components-with-multiple-presentations
disqus_identifer: shared-view-components-with-multiple-presentations
disqus_url: /blog/shared-view-components-with-multiple-presentations
---

ASP.NET Core MVC introduced the concept of [View Components](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/view-components) which are effectively the replacement to child actions of a controller in earlier versions of ASP.NET MVC.  They are useful for encapsulating common components of a web page, such as navigation, recent post lists on a blog and so on.

By design, a view component has one associated view, `/Views/Shared/Components/[component name]/Default.cshtml` (by default its name is `Default.cshtml`) that would render the component's HTML.  However, imagine a case where you want to use the _same_ logic of a component, but display it in a different manner (think of a side bar navigation menu).  This is a common scenario in many of our .NET Core applications.  In an effort to remain DRY, I implemented the ability for our front-end team to specify a different view name for a given component.

Given our example of a main navigation component, let's take a look at what the `MainNavigation` component code might look like:

```c#

public class MainNavigation : ViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync()
    {
        var model = new MainNavigationViewModel();
        model.Items = new List<NavigationItem>();
        
        return View(model);
    }
}

```

As you can see, this is a pretty typical (and simple) component. With the typical implementation, you'd have a matching view, `/Views/Shared/Components/MainNavigation/Default.cshtml` (where "MainNavigation" is the component name and "Default.cshtml" is the standard view filename).

Now, let's see how we might introduce the ability to specify an alternative view, for a "side bar" navigation widget (we'll assume the logic for getting the navigation items is the same for this post).  Update your view component to:

```c#

public class MainNavigation : ViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync(string viewName = null)
    {
        var model = new MainNavigationViewModel();
        model.Items = new List<NavigationItem>();
        
        if (!string.IsNullOrEmpty(viewName)) 
        {
            return View(viewName, model);
        }
        return View(model);
    }
}

```

As you can see, the component is still straight forward.  We added a parameter called `viewName` (I make this optional so it doesn't _have_ to be specified).  After we populate our view model, we add a simple if statement to see if a view name was specified and pass the view name in.

Now, add a new `Sidebar.cshtml` view to your `/Views/Shared/Components/MainNavigation/` directory that has the same model and you're set.

When you want to invoke your component for the side bar, you'd simply specify the view name:

```c#
@await Component.InvokeAsync("MainNavigation", new { viewName = "Sidebar" })

@* If you're using the tag helper syntax... *@
<vc:main-navigation viewName="Sidebar" />

```

And that's it!  Your view component now allows for multiple views! These alternative views for components have helped my team share common logic while allowing the presentation of data to differ.  Feel free to share ideas or feedback in the comments below.