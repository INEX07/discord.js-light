# djs-shenanigans

A modified Discord client built on top of [discord.js](https://github.com/discordjs/discord.js) 12.0.0, Shenanigans aims to drastically reduce discord.js's resource usage while also adding its own set of utilities aimed primarily for reactive message-based bots. This library is very experimental and should be used with caution and lots of testing, as it may disable or break some of discord.js's features.

## Features

djs-shenanigans tweaks discord.js by removing some of its features for performance gains and adding some other features on top.

Pros:

* Drastically lower cpu, memory and network usage (see performance section)
* Automatic sharding with support for multiple processes and multiple shards per process
* On-demand caching (data is cached only when required, say goodbye to sweeping)
* Built-in command handlers with error handling
* Built-in custom prefix handler
* Built-in discordbots.org guild count updater
* Designed to run as replicable independent instances (compatible with pm2 clusters)
* Some additional functions and methods for convenience (see non-standard API section)

Cons:

* Most events are either disabled, handled differently or require additional handling (see non-standard behavior section)
* Presences, typing and guild member events are unavailable due to disabling guild subscriptions (see non-standard behavior section)
* Some features have not been tested (ie: voice)

## Getting Started

Installation:

```npm install timotejroiko/djs-shenanigans```

optional packages (recommended to improve performance, especially zlib-sync. dont use zucc)

```
npm install zlib-sync
npm install bufferutil
npm install discordapp/erlpack (if it doesnt work, use this fork: churchofthought/erlpack)
npm install utf-8-validate
```

Simple usage:

```js
const client = require("djs-shenanigans")(); // good old discord.js client

client.on("message", message => {
	// do stuff
});

client.login("TOKEN")
```

With auto-login, auto-sharding, logging, prefix and command handlers:

```js
const client = require("djs-shenanigans")({
	token:"TOKEN",
	defaultPrefix:"!",
	enableLogger: true,
	enableHandler: "YOURCOMMANDSFOLDER",
	sendErrors:true
});

// create commands in your command folder (see command handlers section)
```

## Client options

All fields are optional.

| Option | Type | Description |
| ------------- | ------------- | ------------- |
| token | string | Your discord bot token. If provided, the client will attempt to negotiate shards and login automatically, else you will need to run client.login() and manually specify shard settings |
| dblToken | string | Your discordbots.org token. If provided, the client will send your guild count to discordbots.org every 24 hours |
| dblTest | boolean | If set to true, the client will also send your guild count to discordbots.org immediatelly after logging in |
| owners | array | Array of user IDs. Used by the non-standard method message.isOwner |
| processes | number | Total number of processes running this bot if running multiple instances manually. Ignored when using pm2 cluster mode |
| process | number | The zero-indexed id of the current process if running multiple instances manually. Ignored when using pm2 cluster mode |
| shardsPerProcess | number | Manually specify the number of shards each process should spawn. Uses recommended shards if omitted or set to "auto" |
| defaultPrefix | string | Default prefix for all guilds and dms |
| customPrefix | function(guildID) | Function that should return a guild-specific prefix from a guild id |
| enableLogger | boolean | Enables logging of connection statuses, messages and errors |
| enableHandler | boolean/string | Command handler mode. See command handlers section |
| enableRoles | boolean | If set to true, role events are enabled and roles and channel permissionOverwrites will be cached (required for permission checking) |
| sendErrors | boolean | If set to true, the command handler will also attempt to send command errors instead of only logging them |

## Command handlers

djs-shenanigans has 3 different ways of operating depending on the command handler mode. The command handler will process all messages from non-bot users that start with a valid prefix or a bot mention, all other messages will be ignored. All bot responses are cached by default. The command handler also listens to message edits and processes them as new messages. You can differentiate between new and updated messages by checking for the existence of message.editedTimestamp. If responding to a message edit using message.send(), the response will be sent as a message edit if the previous response is cached (see non-standard API). When using Router or File mode, empty commands (prefix + nothing or mention + nothing) will be processed as a "nocommand" command.

### Message Event Mode

options.enableHandler set to false or omitted.

In this mode, client will listen to messages that start with a valid prefix and emit them as a "message" event. This is the most barebones setup, messages are captured from the raw event, checked for prefixes and bot users and then emitted. You can also use this event to capture messages from whitelisted channels, regardless of command handler mode (see non-standard API). Example:

```js
const client = require("djs-shenanigans")({
	defaultPrefix:"!"
});

client.on("message", message => {
	// only messages from non-bot users which start with a valid prefix are received (ie: "!somecommand")
	// the message itself is not cached, but its channel and author/member are automatically cached.
	// you must handle commands and errors by yourself
	// also receives messages from whitelisted channels regardless of command handler mode
});
```

### Router Mode

options.enableHandler set to true.

In this mode, client will listen to messages that start with a valid prefix and contain a command registered as an event. This is for those who like the idea of handling commands like a webserver. Registered command events are prefixed with a slash to avoid interfering with standard events. Example:

```js
const client = require("djs-shenanigans")({
	defaultPrefix:"!",
	enableHandler:true
});

client.on("/ping", message => {
	// only messages from non-bot users which start with a valid prefix followed by the command "ping" are received (ie: "!ping")
	// this message, its channel and author/member are automatically cached.
	// you must handle errors by yourself.
});

client.on("/nocommand", message => {
	// empty commands go here (ie: prefix+nothing or mention+nothing)
})
```

### File Mode

options.enableHandler set to a string pointing to a folder.

In this mode, client will listen to messages that start with a valid prefix and contain a valid command file. This is the standard command handler approach, the client will scan the supplied folder and register all files found as commands using their file names. This mode also enables a built-in command reloading function (see non-standard API). Example:

```js
// index.js
const client = require("djs-shenanigans")({
	defaultPrefix:"!",
	enableHandler:"commands"
});

// scans the "commands" folder for js files and registers them in client.commands
// client.commands is a Map object with the file name serving as the key, and its exported object as the value.
```

```js
// commands/ping.js
module.exports.run = message => {
	// only messages from non-bot users which start with a valid prefix followed by the command "ping" are received (ie: "!ping")
	// this message, its channel and author/member are automatically cached.
	// errors that happen inside here are handled and logged automatically. if options.sendErrors is set to true, the error is also sent as a response
	// commands must contain a "run" function, other props such as module.exports.help can be optionally added for management and interaction with client.commands
}
```

```js
// commands/nocommand.js
module.exports.run = message => {
	// empty commands go here (ie: prefix+nothing or mention+nothing)
}
```

## Non-standard API

djs-shenanigans has some extra functions built in for convenience:

| Func/Prop | Returns | Description |
| ------------- | ------------- | ------------- |
| message.send(content,options) | promise>message | This function is the same as message.channel.send() but adds several improvements: can send unresolved promises, objects, falsey values and other non-string types, truncates large strings if no split options are provided, logs response times and sending errors/warnings if logging is enabled, adds request-response pairing if messages are cached and if possible sends responses as edits when triggered by message edits |
| message.asyncEval(string) | promise>anything | An eval function compatible with promises, async/await syntax and complex code. Can access the client via `client` and the message object via `this` (should be locked to owners only) |
| message.isOwner | boolean | Quickly check if the user who sent the message is a bot owner. Uses the array of owners from options.owners |
| message.commandResponse | message | The message object that was sent as a response to this command. Only available if it was sent with message.send() and the message is cached |
| message.commandMessage | message | The message object that triggered this response. Only available if this response was sent with message.send() and the triggering message is cached |
| message.commandResponseTime | number | Message response time in milliseconds. Only available in response messages if they were sent with message.send() and are cached; |
| message.command | string | The command used without prefix and content. Only available with the command handler in router or file mode |
| message.argument | string | The message content without prefix and command. Only available with the command handler in router or file mode |
| channel.whitelisted | boolean | If set to true, this channel will fire "message" events for all messages, instead of only messages that start with a valid prefix or command. Whitelisted channels also fire "messageDelete" and "messageDeleteBulk" events |
| channel.createCollector(filter,options) | messageCollector | The same as channel.createMessageCollector() but whitelists the channel during the duration of the collector |
| client.getInfo() | promise>object | Gather several statistics about the client including all shards and, if running in pm2 cluster mode, all processes. Statistics include total guild count, total user count at login, total active users and channels, websocket pings, uptimes, cpu usage, memory usage and more |
| client.shutdown() | boolean | Begins graceful shutdown in this process, replaces all functions and commands with a temporary message and exits the process after a few seconds |
| client.pm2shutdown() | boolean | Sends a shutdown signal to all processes in the pm2 cluster. Only available when running in pm2 cluster mode |
| client.survey(string) | promise>array | Similar to broadcastEval() but for pm2 clusters. Sends a string to be evaluated by all processes in the cluster and returns an array of responses indexed by process number. Only available when running in pm2 cluster mode |
| client.broadcast(string) | promise>array | Same as client.survey() but it does not wait for a response. It returns an array of booleans representing whether the message was received by the target processes or not. Only available when running in pm2 cluster mode |
| client.commands | map | Where commands are stored when running the command handler in file mode |
| client.commands.reload(command) | boolean | Function to reload a command managed by the command handler in file mode. Can be used to add/re-enable/reload commands without restarting the bot |
| client.commands.disable(command) | boolean | Function to disable a command managed by the command handler in file mode |

## Non-standard behavior

Since this library tampers with discord.js's functions and caches, there is a lot of unexpected behavior, here are a few documented behavior changes from my tests (there might be other untested unexpected behaviors, feel free to contribute with your tests and use cases)

| Reactions | Changes |
| ------------- | ------------- |
| reactions | All message reaction events and collectors should work, but the reaction object is a bit different and might contain partials (a partial is an object that only contains an id and nothing else) |
| reaction.channel | The channel object or channel partial if not cached |
| reaction.message | The message object or message partial if not cached |
| reaction.guild | The guild object or null if DM |
| reaction.user | The user object or user partial if not cached |
| reaction.emoji | The emoji object as per the Discord Gateway API (not the reaction emoji object from discord.js) |

| Channels | Changes |
| ------------- | ------------- |
| client.channels | Channels are cached only when a valid command is used in them, messages are only cached when using the command handler in router or file mode, channel permissions are cached if options.enableRoles is set to true |
| channel.messages | Caches only messages that were processed by the command handler in router or file mode |
| channel.permissionOverwrites | Empty, unless options.enableRoles is set to true. Can also be manually cached by channel.fetch() or client.channels.fetch(id) (roles are required for permission checking functions) |

| Guilds | Changes |
| ------------- | ------------- |
| client.guilds | All guilds are cached and auto-updated by default but not all properties and stores are available |
| guild.channels | Only channels where commands are used are cached |
| guild.members | Only members that used commands are cached. Cached members are updated as they send new messages. Specific members can be cached by guild.members.fetch(id) (fetching all members is disabled) |
| guild.roles | Empty unless options.enableRoles is set to true. Can also be manually cached using guild.fetch() or guild.roles.fetch() (role caching is required for permission checking functions) |
| guild.emojis | Always empty, unless manually cached using guild.fetch() |
| guild.presences | Always empty |
| guild.voiceStates | Always empty |

| Users | Changes |
| ------------- | ------------- |
| client.users | Only users that used valid commands are cached. Specific users can be cached by client.users.fetch(id). Users are updated as they send new messages |

| Events | Changes |
| ------------- | ------------- |
| events | Many events are modified or disabled |
| messageUpdate | The messageUpdate event is fired only in whitelisted channels. Message updates that contain valid commands are processed by the command handler and sent as new messages and contain message.editedTimestamp and a history of changes in message.edits if cached |
| messageDelete / messageDeleteBulk | Message delete events are fired only in whitelisted channels |
| memberUpdate / userUpdate | These events are unavailable as a side effect of disabling guild subscriptions, however, as a workaround, all messages sent by cached users/members will be checked for updated user/member data and will fire userUpdate/memberUpdate events when changes are detected |
| memberCreate / memberDelete | These events are unavailable as a side effect of disabling guild subscriptions. Detecting member joins and leaves is currently not possible without guild subscriptions |
| channelUpdate / channelDelete | These events are enabled only for cached channels |
| roleCreate / roleUpdate / roleDelete | Role events are enabled only if options.enableRoles is set to true (role caching is required for permission checking functions) |
| channelCreate / channelPinsUpdate / emojiCreate / emojiDelete / emojiUpdate / guildIntegrationsUpdate / guildBanAdd / guildBanRemove / webhookUpdate / voiceStateUpdate / presenceUpdate / typingStart | These events are all disabled |

Some disabled events may eventually be re-enabled further down the road when more testing is done.

## PM2 Cluter Mode

djs-shenanigans is compatible with pm2 clusters, all you need to do is run it like this:

```
pm2 start yourFileName.js -i numberOfProcesses --name=yourProcessName
```

To scale your bot, all processes need to be restarted. This can also be done easily with pm2 clusters:

```
pm2 scale yourProcessName numberOfProcesses && pm2 restart yourProcessName
```

When running in pm2 cluster mode, you have access to cluster specific functions such as client.broadcast() client.survey() and client.pm2shutdown().
Cluster mode automatically negotiates shards and spreads them equally across processes, or you can set a specific amount of shards using options.shardsPerProcess.
Client logins are queued using a lockfile to avoid too many login attempts.

## Manual Instances

Running multiple instances manually across a single machine or multiple machines is possible but each instance must be configured with a process id and total processes count in the client options. Sharding is then negotiated automatically. Be aware that login queueing will not be available, so you will need wait for each process to fully login before firing another process to avoid being banned by discord (discord only allows one login every 5 seconds, shards count as logins), as well as use your own inter-process communication if needed. You can also use a master process to control everything like traditional sharders.

## Performance

This test case was done on ubuntu 18 (1vcpu, 1gb ram) running around 1500 guilds with all optional libraries installed (zlib-sync, erlpack, bufferutil, utf-8-validate). Data was recorded using `top` and `nethogs`. The following scripts were used:

```js
// discord.js default settings
const { Client } = require("discord.js");
const client = new Client();
client.login("TOKEN");
```

```js
// discord.js with most things disabled
const { Client } = require("discord.js");
const client = new Client({
    messageCacheMaxSize:0,
    messageCacheLifetime:30,
    messageSweepInterval:60,
    disableEveryone:true,
    disabledEvents:["GUILD_MEMBER_ADD","GUILD_MEMBER_REMOVE","GUILD_MEMBER_UPDATE","GUILD_MEMBERS_CHUNK","GUILD_INTEGRATIONS_UPDATE","GUILD_ROLE_CREATE","GUILD_ROLE_DELETE","GUILD_ROLE_UPDATE","GUILD_BAN_ADD","GUILD_BAN_REMOVE","GUILD_EMOJIS_UPDATE","CHANNEL_PINS_UPDATE","CHANNEL_CREATE","CHANNEL_DELETE","CHANNEL_UPDATE","MESSAGE_CREATE","MESSAGE_DELETE","MESSAGE_UPDATE","MESSAGE_DELETE_BULK","MESSAGE_REACTION_ADD","MESSAGE_REACTION_REMOVE","MESSAGE_REACTION_REMOVE_ALL","USER_UPDATE","USER_SETTINGS_UPDATE","PRESENCE_UPDATE","TYPING_START","VOICE_STATE_UPDATE","VOICE_SERVER_UPDATE","WEBHOOKS_UPDATE"]
});
client.login("TOKEN");
```

```js
// djs-shenanigans
const client = require("djs-shenanigans")({
	token:"TOKEN",
	defaultPrefix:"!",
	enableLogger: true,
	enableHandler: "commands",
	sendErrors:true
});
```

Results:

![CPU Usage](bench/cpu.jpg)
![Memory Usage](bench/mem.jpg)
![Network Usage](bench/net.jpg)

As you can see, djs-shenanigans uses significantly less resources compared to discord.js v12 in this test case. Roughly 5-6x less memory, 5-10x less cpu and 5-20x less network bandwidth.

## About

This project is highly experimental, so the code is quite rough and there might be bugs and broken features especially in untested scenarios (i have tested only features that my bots need). You are encouraged make your own tests with your specific use cases and post any issues, questions, suggestions or contributions you might find.

You can also find me in my [discord](https://discord.gg/BpeedKh) (Tim#2373)

