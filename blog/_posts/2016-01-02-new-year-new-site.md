---
layout: post
title: 'New Year, New Site'
date: 2016-01-02 12:00:00
author: 'Matt Millican'
permalink: blog/new-year-new-site/
disqus_identifier: new-year-new-site
disqus_url: /blog/post/new-year-new-site
redirect_from: /blog/post/new-year-new-site
---

_Note: This blog has since been moved to use Jekyll on GitHub Pages_

In August 2015, I started rebuilding my site using the new ASP.NET vNext framework.  This was actually during thatConference, after I heard a few [more] talks about vNext.  Unfortunately, with vNext being in beta 4 at the time, I felt it was a little too early to really use it.  Not to mention many of the APIs were constantly changing.

Also around the same time (maybe a month before), I got an email from Google Webmaster Tools saying my site was not mobile friendly.  Yup, that was a true statement.  It was time for something new.  Something more current, and cleaner.  

Fast forward to early December, I set a goal that I was going to have my new site launched for the new year.  I wanted it to be in vNext, mainly because I wanted to learn the "new stuff" but also because I thought it [my site] would be a good candidate for it.  I wanted the site to be simple: a small blog, and all other content would be "static" (not CMS driven).  

Here's what my stack looks like:

- ASP.NET vNext (framework version 4.6) & C#
- Entity Framework 7
- Bootstrap 3 (for UI framework)
- Sass for CSS compliation
- Gulp (for minifying CSS and JS)
- Kendo UI (date pickers and other UI elements)
- CKEditor
- Azure Blob Storage (for storing images uploaded with blog posts)

I had to use the full 4.6 framework instead of the CoreClr, because I wanted a contact form and System.Net.Mail had not been ported to CoreClr yet.   Following this issue on Github, it looks like I may be able to soon switch back to the CoreClr.

One of the bigger issues for me was handling URL redirects from my old site to the new.  In older versions of .NET, you could do this using using the web.config `<rewrite>` element, but I wanted something perferably stored in JSON.  I didn't need an admin, since I rarely add new redirects.  I ended up making a small piece of OWIN middleware, which I'll do a how-to of in a future blog post.

Hope you enjoy!  Feel free to contact me or connect on social media.