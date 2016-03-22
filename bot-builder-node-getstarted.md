---
layout: page
title: Getting Started with Bot Builder for Node.js
permalink: /bot-builder-node-getstarted/
weight: 270
parent1: Bot Builder SDK
parent2: Node.js
---


* TOC
{:toc}

## Overview
Bot Builder is a framework for building conversational applications (“Bots”) using Node.js. From simple command based bots to rich natural language bots the framework provides all of the features needed to manage the conversational aspects of a bot. You can easily connect bots built using the framework to your users wherever they converse, from SMS to Skype to Slack and more...

## Installation
To get started either install the BotBuilder module via NPM:

    npm install --save botbuilder

Or clone our GitHub repository using Git. This may be preferable over NPM as it will provide you with numerous example code fragments and bots:

    git clone git@github.com:Microsoft/BotBuilder.git

Examples can then be found under the “Node/examples” directory of the cloned repository. 

## Hello World
Once the BotBuilder module is installed we can get things started by building our first “Hello World” bot called HelloBot. The first decision we need to make is what kind of bot do we want to build? Bot Builder lets you build bots for a variety of platforms but for our HelloBot we're just going to interact with it though the command line so we're going to create an instance of the frameworks TextBot class. 

    var builder = require('botbuilder');

    var helloBot = new builder.TextBot();

We then need to add a dialog to our newly created bot object. Bot Builder breaks conversational applications up into components called dialogs. If you think about building a conversational application in the way you'd think about building a web application, each dialog can be thought of as route within the conversational application. As users send messages to your bot the framework tracks which dialog is currently active and will automatically route the incoming message to the active dialog. For our HelloBot we'll just add single root '/' dialog that responds to any message with “Hello World” and then we'll start the bot listening with a call listenStdin().

    var builder = require('botbuilder');

    var helloBot = new builder.TextBot();
    helloBot.add('/', function (session) {
        session.send('Hello World');
    });

    helloBot.listenStdin();

We can now run our bot and interact with it from the command line. So run the bot and type 'hello':

    node app.js
    hello
    Hello World

## Collecting Input
It's likely that you're going to want your bot to be a little smarter than HelloBot currently is so let's give HelloBot the ability to ask the user their name and then provide them with a personalized greeting. First let's add a new route called '/profile' and for the handler we're going to use something called a waterfall to prompt the user for their name and then save their response:

    var builder = require('botbuilder');

    var helloBot = new builder.TextBot();
    helloBot.add('/', function (session) {
        if (!session.userData.name) {
            session.beginDialog('/profile');
        } else {
            session.send('Hello %s!', session.userData.name);
        }
    });
    helloBot.add('/profile',  [
        function (session) {
            builder.Prompts.text(session, 'Hi! What is your name?');
        },
        function (session, results) {
            session.userData.name = results.response;
            session.endDialog();
        }
    ]);

    helloBot.listenStdin();

By passing an array of functions for our dialog handler a waterfall is setup where the results of the first function are passed to input of the second function. We can chain together a series of these functions into steps that create waterfalls of any length. 

In the first step of the '/profile' dialogs waterfall we're going to call the built-in Prompts.text() prompt to greet the user and ask them their name. The framework will route the users' response to that question to the results value of the second step where we'll save it off and then end the dialog. To save their response we're leveraging the frameworks built in storage constructs. You can persist data for a user globally by assigning values to the session.userData object and you can also leverage more temporary per/dialog storage using session.dialogData.

To make use of our new '/profile' dialog we also had to modify our root '/' dialog to conditionally start the '/profile' dialog. The root '/' dialog now checks to see if we know the users name and if not it redirects them to the '/profile' dialog using a call to beginDialog(). This will execute our waterfall and then control will be returned back to the root dialog with a call to endDialog(). We can now run HelloBot again to see the results of these improved smarts:

    node app.js
    hello
    Hi! What is your name?
    John
    Hello John!

## Handling Commands
So far we've shown the creation of dialogs based on closures but the framework comes with a number of classes that can be used to create more sophisticated dialogs. Let's use the CommandDialog class to add a couple of commands that make our bot a little more useful.  The CommandDialog lets you add a RegEx that when matched will invoke a Dialog Handler similar to the ones we've been creating so far. We'll add a command for changing the name set for our profile and then a second command to let us quit the conversation.

    var builder = require('botbuilder');

    var helloBot = new builder.TextBot();
    helloBot.add('/', new builder.CommandDialog()
        .matches('^set name', builder.DialogAction.beginDialog('/profile'))
        .matches('^quit', builder.DialogAction.endDialog())
        .onDefault(function (session) {
            if (!session.userData.name) {
                session.beginDialog('/profile');
            } else {
                session.send('Hello %s!', session.userData.name);
            }
        }));
    helloBot.add('/profile',  [
        function (session) {
            if (session.userData.name) {
                builder.Prompts.text(session, 'What would you like to change it to?');
            } else {
                builder.Prompts.text(session, 'Hi! What is your name?');
            }
        },
        function (session, results) {
            session.userData.name = results.response;
            session.endDialog();
        }
    ]);

    helloBot.listenStdin();

To add commands we changed our root '/' dialog to use an instance of a CommandDialog which you can see uses a fluent style interface to configure. We moved our existing dialog handler to become the onDefault() behavior of the dialog and we add our two commands. We're using DialogActions to implement the commands which are simple shortcuts that create a closure for a common action. The beginDialog() Dialog Action is going to begin the '/profile' dialog anytime the user says “set name” and the endDialog() action will exit the conversation when the user says “quit”. We also tweaked our '/profile' prompt to say something slightly different when changing the users name.  If we now run our updated HelloBot we get:

    hello
    Hi! What is your name?
    John
    Hello John!
    set name
    What would you like to change it to?
    John Smith
    hello
    Hello John Smith!
    quit
 
## Publishing to the Bot Connector Service
Now that we have a fairly functional HelloBot (it does an excellent job of greeting users) we should publish it the Bot Connector Service so that we can talk to it from within various communication apps. Code wise we'll need to first switch to using a BotConnectorBot instead of a TextBot:
 
    var restify = require('restify');
    var builder = require('botbuilder');
    
    var server = restify.createServer();

    var helloBot = new builder.TextBot();
    helloBot.add('/', new builder.CommandDialog()
        .matches('^set name', builder.DialogAction.beginDialog('/profile'))
        .matches('^quit', builder.DialogAction.endDialog())
        .onDefault(function (session) {
            if (!session.userData.name) {
                session.beginDialog('/profile');
            } else {
                session.send('Hello %s!', session.userData.name);
            }
        }));
    helloBot.add('/profile',  [
        function (session) {
            if (session.userData.name) {
                builder.Prompts.text(session, 'What would you like to change it to?');
            } else {
                builder.Prompts.text(session, 'Hi! What is your name?');
            }
        },
        function (session, results) {
            session.userData.name = results.response;
            session.endDialog();
        }
    ]);

    server.use(helloBot.verifyBotFramework({ appId: 'you id', appSecret: 'your secret' }));
    server.post('/v1/messages', helloBot.listen());

    server.listen(8080, function () {
        console.log('%s listening to %s', server.name, server.url); 
    });

Our updated bot code now pulls in Restify which we'll use to setup the skeleton of HelloBots’ web service. We then created a new BotConnetcorBot instead of a TextBot and finally we wired up the bots listener() to a route off the server. For security reasons its recommended that you lock your server down to only receive requests from the Bot Connector Service so we can call verifyBotFramework() to install a piece of middleware that does that.

You can then use the Bot Framework Emulator to locally test your changes and once you’ve verified that everything works you can deploy your bot to your hosting service of choice and then go through the steps below to register your bot with the Bot Connector.

## Registering your Bot with the Connector
Registering your Bot tells the Connector how to call your Bot's web service. Note that the AppId and AppSecret are generated when your Bot is registered with the Microsoft Bot Framework Connector, the AppId and AppSecret are used to authenticate the conversation, and allows the developer to configure their Bot with the Channels they'd like to be visible on.

1. Go to the Microsoft Bot Framework portal at [https://www.botframework.com](https://www.botframework.com) and sign in with your Microsoft Account.
	
2. Register an agent
	
3. Click the "Register a Bot" button and fill out the form. Many of the fields on this form can be changed later. Use a dummy endpoint for now. Save your changes by hitting "Create" at the bottom of the form, and don't worry about the other settings for now.

![Register a bot](/images/connector-getstarted-register-agent.png)

4. Once your registration is created, Microsoft Bot Framework will have generated your AppId and AppSecrets. These are used to authenticate your Bot with the Microsoft Bot Framework, and will be used in the next step.

![Microsoft Bot Framework will have generated your AppId and Subscription Keys](/images/connector-getstarted-subscription-keys.png)