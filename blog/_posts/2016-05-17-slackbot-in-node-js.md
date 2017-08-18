---
layout: post
title: 'Build a Slackbot in Node JS'
subtitle: 'Build a Slackbot using Node JS and Botkit'
date: 2016-05-17 12:00:00
author: 'Matt Millican'
permalink: blog/build-a-slackbot-in-node-js/
disqus_identifier: build-a-slackbot-in-node-js
disqus_url: blog/post/build-a-slackbot-in-node-js
redirect_from: blog/post/build-a-slackbot-in-node-js
---

Our development team recently made the switch from Hipchat to [Slack](http://slack.com) for team communication.  Previously, we had a few integrations with other tools we use, such as BitBucket and Bamboo.  There were also a few new ones that we wanted, like the one I'm going to walk you through today.

Back story: Over the past year or so, I've been working on an [agile] project management system, that my team is currently beta testing.  We often reference tasks (and user stories, etc) in chat.  This usually involves a user finding the task and then copying and pasting the URL in chat.  This is fine and all, but for such a repetitive task, I thought it helpful to have a bot do this for us.  A sample message to "collabot" might look like: `@collabot: get task 2016-01-0101`, where "2016-01-0101" is the task number.

## Setting up the integration

To start, you need to create your integration for your new bot.  To do so, go to _http://<team_id>.slack.com/apps/build_ and choose if you want to develop for "everyone" or "just your team."  For mine, I chose just my team.  As you can see, there are a few types of integrations one could build for slack.  For our purposes, we'll choose "bot."

![Slack integration options](https://m2uploads.blob.core.windows.net/prod/slack-integration-options.png)

When you click on "bot," you'll be asked to create a name for your bot.  For mine I did "collabot" (it's a play on the name of our agile system).  Note that you can change this later, and I did a few times.  From there, you'll be taken to the final screen where you'll see your API token (this is important, but keep it safe).  You can also change your bot's display name, and icon.

## Setting up the project

For purposes of this post, I'm assuming you have [Node.js](https://nodejs.org/en/) and all the related necessities (NPM etc) installed and ready to develop.

In a command prompt, where you'd like your app, run

```
> npm init
```

to initialize your new project.  Node will walk you through a series of questions to get your project set up.  Default options will show in parenthesis, and at the end, it will show you what your package.json will look like.  You can always change these later by editing the package.json file.

Next, we need to include two NPM packages for our bot:

```
> npm install botkit --save
> npm install request --save
```

"[Botkit](https://www.npmjs.com/package/botkit)" is the library I chose to assist in interacting with Slack's APIs and "request" is used for making the HTTP requests to the project management app's API.  Be sure to include "--save" so that the package gets added to your project correctly.

## Creating the bot

Now, for the fun part -- the code.  The majority of the magic happens in `collab-bot.js`.  I'll highlight the main parts of it, but leave the rest up to you, since each use-case will likely be different.

``` js
var Botkit = require('botkit');
var request = require('request');

var controller = Botkit.slackbot({});

var CollabBot = function Constructor(settings){
    this.settings = settings;
    this.settings.name = this.settings.name || 'collab-bot';
};
```

Here, we're essentially using require to bring in our dependendencies, and then setting up a controller for Botkit.  Finally, a simple constructor that our app will call and pass in the settings for our bot.

``` js
CollabBot.prototype.run = function() {
    var self = this;

    controller.spawn({
        token: this.settings.token
    }).startRTM();

    controller.hears('get task (.*)', ['direct_mention'], function(bot, message) {
        var taskId = message.match[1];

        // Define the "success" callback which will be passed into _getTask()
        var callback = function(task) {
            if (!task) {
                return;
            }

            // Add your data from the API response here, and then use this as your reply
            var replyText = 'I found the task: ' + task.title;

            bot.reply(message, replyText);
        };

        // Define the "error" callback, which will be passed into _getTask()
        var errorCallback = function(errorMsg) {
            bot.reply(message, errorMsg);
        };

        // Redacting this method for brevity
        // TL;DR: It uses 'request' to call my API and retrieve task data
        self._getTask(taskId, callback, errorCallback)
    })
};
```

The run method is where [most of] the magic happens.  To start, we're telling our controller (defined above) to "spawn" an instance, passing in our API token via settings.  Then we're using `.startRTM()` to use Slack's Real Time Messaging API.

Next, we want to tell the bot what to listen for.  In my case, I only want it listen to direct mentions.  I am using a regular expression to make sure the message starts with "get task" and then I'm using the last part as the task ID (this looks something like "2016-01-0123").  Because the AJAX call is asynchronus, and for readability, I'm defining my callback method which calls my API (in a later method) and then formats the message using the response from the API (redacted) and sends it to `bot.reply(message, replyText)`.  For purposes of brevity, I'm not including any of my code to make the API calls for getting data, but all of that happens in `_getTask()`.

I end the file by exporting CollabBot:

``` js
    module.exports = CollabBot;
```

## Setting up to run

The final piece of code we have to write is `index.js` which is responsible for telling Node how to run the bot (the collab-bot.js file).  This file is pretty straightforward here goes:

``` js
'use strict';

var CollabBot = require('./collab-bot');

var settings = {
    token: '<your-slackbot-token>',
    collab: { 
        apiUrl: 'http://localhost:1444/api/',
        appUrl: 'http://localhost:1444/',
        username: '<username>',
        token: '<api-token>'
    }
};

var collabBot = new CollabBot(settings);
collabBot.run();
```

To start, we're simply using require to "include" CollabBot.  From there, we're defining the settings our bot needs to run.  Most of these are for my API specfiically, but the required one for any bot will be the `token`, which is your Slackbot's API token (from above).

I'm then instantiating a new `CollabBot`, passing in my settings object, and then calling `.run()`.

## Conclusion

Overall, the process was pretty straightforward.  My colleagues and I have been talking about many other ideas for bots to built as well as various integrations.  Although not everyone agrees, Bots do make things a little faster and simpler, as they enable you to automate repetitive tasks.  Have fun bot-ing!  Share what other bots you've made (or use) in the comments below.

PS.  Interested in Collab?  [Get in touch](/contact) to see if you're a good fit for beta testing!