---
layout: post
title: 'Page Level Open Graph Data in Sitefinity'
subtitle: "Add open graph tags on pages in Sitefinity"
date: 2018-05-04 12:00:00
author: 'Matt Millican'
# header-img: 'img/blog/scaffolding.jpg'
permalink: blog/page-open-graph-data-in-sitefinity
---

[Open Graph](http://ogp.me/) enables content on web pages to be presented in a more sharable format for 
social networks such as Facebook and Twitter.

Out of the box, [Sitefinity](https://www.sitefinity.com/) allows you to create custom fields for certain widgets such as blog posts, events and news items to [display open graph data](https://docs.sitefinity.com/open-graph-settings) on their details pages. Unfortunately, if you want to create OG data at the _page_ level, this isn't quite as easy, but still possible. Special props goes to fellow TDE [Steve](https://www.sitefinitysteve.com/) for helping out with this (as well as many other bumps along the way). 

## Creating the Custom Fields

To start, we need to create the custom fields for Pages. Within the backend, navigate to "Pages" and click on "Custom Fields" under "Settings for pages" in the right column. You should see something similar to the below, depending if you have existing custom fields or not:

![Sitefinity page custom field list](/img/blog/sitefinity/sitefinity-page-custom-fields-list.png)

The three fields we will be creating here are `OpenGraphTitle`, `OpenGraphDescription` and `OpenGraphImage`. Creating the fields themselves is relatively simple process so instead I will share the settings for each:

- `OpenGraphTitle`: Type is "Short text"
- `OpenGraphDescription`: Type is "Long text", but make sure to change the interface widget to "Text area"
- `OpenGraphImage`: Type is "Related media", kind "images". After clicking "Continue," be sure to click the "Limitations" tab and select "Only 1 image can be uploaded or selected.

Once you have your fields created, click "Save changes" on the page.

## Adding the Open Graph data to pages

Now that the custom fields are created, you can add the data to the pages. The custom fields appear on the Page's "Title & Properties" screen, near the bottom.

> The "Title & Properties" screen is accessible by clicking the "Actions" drop down in the page list or in the upper-right corner of editing a page. Depending on other custom fields you may have defined, the screen should look something like this:

![Sitefinity page custom field editing](/img/blog/sitefinity/sitefinity-page-custom-fields-editor.png)

Enter your social-friendly data as appropriate and save your changes.

## Outputting the meta tags

Open graph data is output in the `meta` tags of a page. Because Sitefinity does not support this out of the box (as of version 10.2), you'll have to edit your `layouts\default.cshtml` file to include them.

Within the `<head>` of the page, I added the following code:

```
@using Telerik.Sitefinity.Web;

@{ 
    var currentNode = SiteMapBase.GetActualCurrentNode();
}

@if (!SystemManager.IsDesignMode && currentNode != null)
{
    var ogTitle = (currentNode.GetCustomFieldValue("OpenGraphTitle") as string) ?? currentNode.Title;
    var ogDescription = (currentNode.GetCustomFieldValue("OpenGraphDescription") as string) ?? currentNode.Description;

    <meta property="og:title" content="@ogTitle" />
    <meta property="og:description" content="@ogDescription" />

    var ogImage = currentNode.GetCustomFieldValue("OpenGraphImage") as Telerik.Sitefinity.Libraries.Model.Image;
    if (ogImage != null)
    {
        <meta property="og:image" content="@ogImage.ThumbnailUrl" />
    }
}
```

To start, we're getting the current page node from the site map and then checking that we are not in design mode and that the page exists. 

Following, we need to retrieve the custom data value and cast it as a string (at least for the title and description). If that's null, we want to coalesce to use the page title (or description) and then output the meta tag.

The image is a little different, but still easy. Instead of casting to a string, you'll want to cast it to `Telerik.Sitefinity.Libraries.Model.Image`. Here, there's not necessarily anything to coalesce, so we'll check to make sure its not null before rendering the `<meta>` tag for it. Because these images being shared are usually presented in a smaller format, I opted to use the `ThumbnailUrl`.

So there you have it - you can now enable open graph data on your Sitefinity pages. If you'd like, you could easily add additional OG properties that are supported. Special thanks again to ["Sitefinity" Steve](https://www.sitefinitysteve.com/) for helping out.