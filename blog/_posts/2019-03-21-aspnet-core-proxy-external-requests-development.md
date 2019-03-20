---
layout: post
title: 'Proxy External Requests During Development in ASP.NET Core'
subtitle: "Use Ngrok to proxy external requests to ASP.NET Core for testing"
date: 2019-03-21 12:00:00
author: 'Matt Millican'
header-img: 'img/blog/abstract-servers-internet.jpg'
permalink: blog/:slug
---

Developing applications involving webhooks or Slack integrations that expect incoming HTTP requests can be tricky. In most cases, your development machine is either behind a home router or firewall or in the corporate world, multiple layers of firewalls and other network devices. This makes testing these types of applications tricky, short of having to deploy to a server each time you want to test a change.

Fortunately, there are services such as [ngrok](https://ngrok.com/) which make exposing your local development server to the outside world, without firewall changes or getting IT involved. Let's take a look at getting this set up.

To start, you'll need an account on ngrok. They offer a free version and paid versions which offer additional features such as multiple tunnels, custom URLs and domains, better security and higher request limits. For now, I'm just using the free version, but I can definitely see upgrading at some point. Once you have an account, there's just a [few simple steps](https://dashboard.ngrok.com/get-started):

1. Download
1. Unzip, and add to your `path`
1. Connect to your ngrok account
1. Start it up

Once downloaded and unzipped, I put the ngrok app in a "global" `_tools` folder and added that to my path for easy access. Alternatively, you can navigate to its residing folder and start from there each time. Next is adding your account key, which is also super easy, and the script is right on the "Get started" page, with your token pre-filled.

Ngrok is all set up and ready to use. Starting it up is as simple as running:

```
$ ngrok http 80
```

where `80` is the port number you want to proxy _to_. If you're using Kestrel, that would be `5000` (or `5001` for HTTPS).

## Forwarding HTTP Headers

If you're using the HTTPS redirection middleware in ASP.NET Core, there's one more change you'll have to make to get things to work correctly. Because Kestrel is behind ngrok and traffic is being forwarded over HTTP (not HTTPS), Kestrel will see the request as HTTP, even though you're using the HTTPS URL from ngrok, and keep trying to redirect your request. 

Fortunately, this is easy in ASP.NET Core and requires just two modifications to `Startup.cs`.

```c#
public void ConfigureServices(IServiceCollection services)
{
    // Your other services

    services.Configure<ForwardedHeadersOptions>(options =>
    {
        options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseForwardedHeaders();

    // Your other middleware registrations
}
```

Now you should be able to rebuild your project and refresh your ngrok URL to start testing.

Hopefully, by using ngrok you'll be able to more easily test your APIs and webhooks without having to worry about firewall rules other networking configurations.