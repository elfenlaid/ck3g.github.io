---
layout: post
title: "Subscribe to Rails ActionCable Channels from mobile clients such as iOS and Android"
desc: "Learn how to subscribe to Rails ActionCable Channels from mobile clients such as iOS and Android"
keywords: "Rails, Ruby on Rails, ActionCable, Action Cable, Web Sockets, WebSockets, Chat, Mobile, iOS, Android"
tags: [Rails, ActionCable, WebSockets]
---

You have decided to write a mobile chat app and use Ruby on Rails with ActionCable as a backend. You have followed ActionCable tutorials and succeeded to build a web-based prototype for that. But your mobile applications cannot connect to the existing channels. Sounds familiar? I had exactly the same issue.

One of the specifics of the ActionCable is its channels. They look pretty simple once you follow examples of ActionCable and try to subscribe to channels from the same application via JavaScript.
It's become a little bit tricky once you decide to use a mobile phone as a client for the ActionCable.
By default WebSocket, mobile libraries for the iOS/Android do not support subscription to the channel.
So you need either find libraries which are built to support ActionCable or implement that on your own.

> I think I owe you a small explanation here. Things may be changed since that article is published.
> But back in the day, we were using [Starscream](https://github.com/daltoniam/Starscream) on the iOS side.
> And now there may be new libraries which provide support of ActionCable and some parts might be done more easily.

Luckily it ended up more easy to do then it seemed to be at the beginning when I start to look for a solution. Since I had almost no knowledge in the WebSockets I had to go through different articles to get fundamental knowledge about the subject. At the end, I’ve dove into the source code ActionCable to understand exactly how to subscribe to the channels.

First of all, you need to establish WebSocket connection with the ActionCable URI. Then, to subscribe to the particular channel it’s enough to send the subscribe command through the socket.

ActionCable can receive command in the format:

```json
{
  "command":"subscribe",
  "identifier":"{\"channel\":\"ChatChannel\"}"
}
```

You should get back a message like this:

```json
{
  "identifier":"{\"channel\":\"ChatChannel\"}",
  "type":"confirm_subscription"
}
```

That would mean you are good to go.

It might be not obvious at the first glance, but ActionCable wants `identifier` to be a string which looks like JSON `{ "channel": "ChannelNameChannel" }`.
The name of the channel goes from the channel class name in the Ruby on Rails project.

Happy Messaging!
