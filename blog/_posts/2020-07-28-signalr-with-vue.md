---
layout: post
title: 'Add SignalR to a Vue.js app'
subtitle: "Add real-time communication to your Vue.js app with ASP.NET Core SignalR"
date: 2020-07-28 00:00:00
author: 'Matt Millican'
header-img: 'img/blog/real-time-dashboard.jpg'
permalink: blog/:slug
---

Adding real-time communication to your web applications such online games, 
dashboards, and chat systems can greatly enhance the usability and interactivity 
of the app. Using WebSockets, your users see the most up-to-date information 
without having to manually refresh their browsers, or your application having 
to poll constantly for changes. 

ASP.NET Core SignalR provides an API to allow server-to-client communication. By 
default, SignalR will use WebSockets but will gracefully fallback if the browser 
doesn't support it. SignalR allows you to easily send messages to all clients at 
once, or send messages to individual users or groups.

In this post, we'll look at how to get SignalR set up, and then how you can add 
it to your Vue.js application.

## Setting up SignalR

For purposes of this post, I'm going to assume you have an API project already
set up. Adding SignalR in an ASP.NET Core project is fairly straight forward. If you 
are targeting `netcoreapp3.1`, SignalR is included in the "meta" package and
there is nothing else to install.

You'll need to set up a "Hub" which is responsible for handling the socket
connections with the clients. These can include both sending and receiving messages. 
One drawback to the traditional hubs is that they often have a lot of "magic strings." 
These are often fragile and can lead to runtime errors. However, Hubs can be strongly-typed to
help mitigate that risk. For purposes of this post, we'll use a strongly-typed hub, but 
it is not required. 
[Read more about strongly-type hubs](https://docs.microsoft.com/en-us/aspnet/core/signalr/hubs?view=aspnetcore-3.1#strongly-typed-hubs).

When using a strongly-typed hub, you'll need to create an interface that represents the 
client. Like any other interface in C#, it represents that contract of the messages and 
their parameters that can be sent. Below is an example of my `IGameClient`:

```c#

public interface IGameClient
{
    Task GameStarted(string gameCode);
    Task PlayerJoined(string playerName); 
}

```

Now that we have our client interface, we need to create a Hub. Here's what that looks
like:

```c#

public class GameHub : Hub<IGameClient>
{
    private readonly IRepository<Game> _gameRepository;
    private readonly IRepository<Player> _playerRepository;

    public GameHub(IRepository<Game> gameRepository,
        IRepository<Player> playerRepository)
    {
        _gameRepository = gameRepository;
        _playerRepository = playerRepository;           
    }

    public async Task GameCreated(string gameCode)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, gameCode);
    }

    public async Task PlayerJoined(string gameCode, string playerName)
    {
        var game = await GetGameByCode(gameCode);
        var player = new Player
        {
            GameId = game.Id,
            Name = playerName,
            JoinedOn = DateTime.UtcNow,
            IsHost = false
        };

        await _playerRepository.CreateAsync(player);

        await Groups.AddToGroupAsync(Context.ConnectionId, gameCode);
        await Clients.OthersInGroup(gameCode).PlayerJoined(playerName);
    }
}

```

You'll notice that the `GameHub` class inherits from `Hub<IGameClient>` - 
this is what's strongly typing our hub to the interface we defined earlier. 
If we were not making a strongly-typed hub, we would just inherit from `Hub` instead.

You'll also see that we have a constructor, with a few repository dependencies. 
A hub can have dependencies injected into it, just like any other controller or 
service. This is helpful if you need to persist data to a database or other 
store.

Following, we have two methods, which coorelate to our `IGameClient` interface. 
We'll look at the `PlayerJoined` method. First, we're getting the game info by 
it's code (method redacted for brevity) and then we are creating a `Player` object 
and persisting it to our data store (in this case that's CosmosDB). SignalR has 
the concept of [groups](https://docs.microsoft.com/en-us/aspnet/core/signalr/groups?view=aspnetcore-3.1#groups-in-signalr), 
so we are adding this player to the group for this game and finally notifying all 
of the other players that the player has joined.

The last thing we need to do for now is to wire up our Hub in `Startup.cs`. Add 
the following line to the `ConfigureServices` method. I typically like to do 
this just after `services.AddControllers()` (or MVC):

```c#
services.AddSignalR();
```

Finally, we need to add the SignalR endpoint, by modifying `UseEndpoints` to look 
like this:

```c#

app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();
    endpoints.MapHub<GameHub>("/gamehub"); // The URL passed can be named whatever you'd like
});
```

## Adding SignalR to Vue

Adding SignalR to Vue was a little trickier than the server-side, at least for me. 
I wasn't sure how to get started, such as "where do I start the connection?" And 
"where do I listen for these messages?" After some brief Google searches and 
reading a few blog posts, I found a package on NPM, 
[vue-signalr](https://www.npmjs.com/package/@latelier/vue-signalr). The 
documentation makes it a little hard to get started so it look a little playing 
around to get things figured out. (Ps once I figure things out more clearly, I 
plan on submitting a PR to update the docs.)

Once you've installed the package, open up `main.js` and add the following:

```javascript
import VueSignalR from '@latelier/vue-signalr'

Vue.use(VueSignalR, 'https://localhost:5001/gamehub')
```

The URL shown above is the URL to your API (or backend project) with the 
URL you defined as the endpoint for the hub. 

Next, you'll need to start the socket connection. For now, I chose to do this 
in my `App.vue` component, but I may move it as I think it is better suited in 
a more "scoped" component (`App.vue` is used as the "base" for all Vue URL routing). 

```javascript
export default {
  created () {
    this.$socket.start({
      log: true // Logging is optional but very helpful during development
    })
  },
  sockets: {
    PlayerJoined (data) {
      // this has to be here even though it's not being used?
      // otherwise other events don't actually fire
    }
  }
}
```

The Vue library adds an additional option, `sockets` that you can add to each 
component to get access to the socket events that can be triggered. The 
alternative to this would be:

```javascript
this.$socket.on('PlayerJoined', (data) => { });
// or
Vue.prototype.$socket.on('PlayerJoined' (data) => { });
```

One caveat I learned here (a very important one) is that you'll need to define 
whatever socket messages you'll be listening for in this file. They can be blank 
as mine is above, but if they aren't there, none of the components seem to get 
the messages. The names of these messages match the names of the methods defined 
in your Hub. 

Now that the setup is out of the way, we can start listening to socket messages 
for the events we care about and update our UI accordingly. This is fairly 
straightforward. You can use either syntax I noted above to do this; I'll show 
the one I'm primarily using in my app:

```javascript
// additional component code redacted for brevity
created () {
    const that = this

    this.$socket.on('PlayerJoined', function (data) {
        that.$store.dispatch('game/playerJoined', { playerName: data })
    })
},
```

Here, we are listening for the `PlayerJoined` message and then dispatch an 
action to our Vuex store with the player's name. We could similarly directly 
update our UI instead of dispatching an action.

Sending messages (or "invoking") is similar to receiving, just a slightly 
different syntax. In the following example, I am redirecting the user to a new 
page after the invocation is successful. You could easily submit a follow-up 
request or prompt the user for more information. Ideally, if you are using Vuex, 
you might want to invoke the request as an action on your store, instead of 
directly from the component.

```javascript
// additional component code redacted for brevity
methods: {
    joinGame () { // triggered by a form submit
        this.$socket.invoke('PlayerJoined', data.gameCode, data.playerName).then(() => {
            this.$router.push(`/game/${this.gameCode}`)
        })
    }
}
```

## Summary

And there you have it; you can now enhance your Vue app's connectivity by 
adding SignalR WebSockets. I am currently trying to figure out how I can 
better integrate the sockets with my Vuex store, so that I don't have to 
listen for messages in components just to call an action on a store. Do you 
use SignalR or other sockets in your Vue application? Have any tips for things 
we could improve here or better integrate with Vuex? Leave a comment below!

<span>Header photo by <a href="https://unsplash.com/@kmuza?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText" target="_blank">Carlos Muza</a> on <a href="https://unsplash.com/s/photos/web-dashboard?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText" target="_blank">Unsplash</a></span>