---
layout: post
title: '.NET MVC + Angular JS: Configuration variables'
date: 2015-01-08 12:00:00
author: 'Matt Millican'
permalink: blog/mvc--angular-js-configuration-variables/
disqus_identifier: mvc--angular-js-configuration-variables
disqus_url: /blog/post/mvc--angular-js-configuration-variables
---

When building client side, single page applications (SPAs), you'll likely need some configuration variables from the server side at some point.  Things such as the app root URL (for linking to other pages within the app), the API URL (interacting with your HTTP API), current user ID, etc.  I found what what seems to be a good way to do this that I have been using in a few of my Angular apps that combines partial/child actions in MVC along with a factory/service in Angular.

The first part is create a child action (or partial view) in MVC that will render the configuration values on the page (but not actually visible to the user).  This involves a child action (returning a PartialViewResult), a simple view model and a partial view.  Since I "host" my Angular app within an MVC View, I can easily leverage these partial view actions.

First, let's create the view model that will have our various config variables.  These are the ones I normally use, but you can put whatever you feel is helpful.

``` c#
public class ConfigViewModel
{
    public string AppBaseUrl { get; set; } 
    public string ApiBaseUrl { get; set; }
    public bool IsAuthenticated { get; set; }
    public Guid? UserId { get; set; }
}
```

Next, we'll add an action to our controller to populate the variables and return the partial view.  I normally put this method on the same controller that's "hosting" the Angular app.

``` c#
[ChildActionOnly]
public PartialViewResult Config()
{
    var model = new ConfigViewModel();
    // set your config variables

    return PartialView("_Config", model);
}
```

Let's create our partial view, "_Config.cshtml" to render the Angular factory.

``` html
@model Models.ConfigViewModel

<script>
    angular.module('myApp').factory('config', function() {
        return {
            appBaseUrl: '@Url.Root()',
            apiUrl: '@Url.Root()api/',
            userId: 'your-user-id'
        }
     });
</script>
``` 

The final thing to do is add the partial view to your MVC "host" page.  

``` html
@Html.Action("Config", "Home")
```

Using this config is quite easy:  Add "config" as a dependency in your Angular controllers (or other services), and you can do `config.apiUrl`.

So far this has been very helpful for me and saves a lot of possible code duplication.  If you have something similar you use, please feel free to share in the comments.