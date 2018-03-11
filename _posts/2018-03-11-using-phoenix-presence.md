---
layout: post
title: "Using Phoenix Presence"
desc: "Using Phoenix Presence to implement list of online users and display typing indicator."
keywords: "Elixir, Phoenix, Channels, WebSockets, Presence"
tags: [Elixir, Phoenix, Channels]
excerpt_separator: <!--more-->
---

In [the previous article](http://whatdidilearn.info/2018/03/04/using-channels-in-phoenix.html) we have implemented chat functionality.
Our users can join a room and start sending messages to a room.
By implementing that we have covered very basic functionality of Phoenix channels.

Today I would like to go deeper and take a look at Presence feature, which ships with Phoenix since version 1.2.
In this article, we will use Presence functionality to implement a list of online users and render the typing indicator next to a username.

<!--more-->

## What is Phoenix Presence

As [official documentation](https://hexdocs.pm/phoenix/presence.html) says:

> Phoenix Presence is a feature which allows you to register process information on a topic and replicate it transparently across a cluster. It’s a combination of both a server-side and client-side library which makes it simple to implement. A simple use-case would be showing which users are currently online in an application.

If that sounds scary for now, don't worry, we will figure out during the implementation. Let's start.

## Show online users

To start using Presence in a Phoenix project, first, we need to create a module and extend it with the presence functionality.
We can either do it manually or use the following generator and let Phoenix do the dirty work for us.

```
→ mix phx.gen.presence
```

That command generates a `lib/prater_web/channels/presence.ex` file with the following content.

```elixir
defmodule PraterWeb.Presence do
  use Phoenix.Presence, otp_app: :prater,
                        pubsub_server: Prater.PubSub
end
```

Actually, it also contains a huge `@moduledoc`, but I've omitted it here.

As a next step, we need to add our `Presence` module into a supervision tree.
Open `lib/prater/application.ex` and update list of `children` to contain it.

```elixir
children = [
  # ...
  supervisor(PraterWeb.Presence, [])
]
```
Then we will move to our `RoomChannel` module and extend it.

```elixir
alias PraterWeb.Presence

def join("room:" <> room_id, _params, socket) do
  send(self(), :after_join)
  {:ok, %{channel: "room:#{room_id}"}, assign(socket, :room_id, room_id)}
end
```

We are adding the alias for `Presence` module so we can use shorter name later.
Once a user has been joined a channel we are sending a message to a current process with `:after_join` atom as a content.

You can review how to [work with processes in Elixir](http://whatdidilearn.info/2017/12/17/elixir-multiple-processes-basics.html) and check [introduction to OTP](http://whatdidilearn.info/2018/01/07/introduction-to-otp-genservers-and-supervisors.html).

Next step would be to catch the message using `handle_info/2` function.

```elixir
def handle_info(:after_join, socket) do
  push socket, "presence_state", Presence.list(socket)

  user = Repo.get(User, socket.assigns[:current_user_id])

  {:ok, _} = Presence.track(socket, "user:#{user.id}", %{
    user_id: user.id,
    username: user.username
  })

  {:noreply, socket}
end
```

First, we are pushing a new "presence_state" message to a socket. Then we are asking `Presence` to track the presence of the user and pass additional data, which we will be able to fetch soon. By doing that our presences data will contain information in the following format:

```json
{
  "user:2": {
    "metas": [
      {
        "username":"user",
        "user_id":2,
        "phx_ref":"Pw0hn5w3Igw="
      }
    ]
  },
  "user:13": {
    "metas": [
      {
        "username":"user503",
        "user_id":13,
        "phx_ref":"DaxoR0uNDSw="
      }
    ]
  }
}
```

Where `user:n` is a topic name we are setting and `metas` contains additional information.

That's it on back-end side. Now we can move to the front-end side.

There is only one change we need to make in HTML is to add a container which will keep the list of our online users.

Let's add the following markup:

```html
<div class="col-md-3">
  <div class="card">
    <h5 class="card-header text-white bg-secondary">Online:</h5>
    <div class="card-body">
      <div id="online-users"></div>
    </div>
  </div>
</div>
```

to the `lib/prater_web/templates/room/show.html.eex` file.

Rest of the work we are going to do in the `assets/js/socket.js` file.

On top of that file, change:

```js
import {Socket} from "phoenix"
```

to

```js
import {Socket, Presence} from "phoenix"
```

We also need to set the initial value for `presences` variable, which we are going to use in a moment.

```js
let presences = {};
```

```js
channel.on("presence_state", state => {
  presences = Presence.syncState(presences, state)
  renderOnlineUsers(presences)
})

channel.on("presence_diff", diff => {
  presences = Presence.syncDiff(presences, diff)
  renderOnlineUsers(presences)
})
```

Here we are listening to two events: "presence_state" is fired once a user has been joined the channel and "prensece_diff" are raised every time someone joins or leaves the channel.

In both cases, we are syncing the changes and pass them to `renderOnlineUsers` function, which will render the list of users for us.

```js
const renderOnlineUsers = function(presences) {
  let onlineUsers = Presence.list(presences, (_id, {metas: [user, ...rest]}) => {
    return onlineUserTemplate(user);
  }).join("")

  document.querySelector("#online-users").innerHTML = onlineUsers;
}

const onlineUserTemplate = function(user) {
  return `
    <div id="online-user-${user.user_id}">
      <strong class="text-secondary">${user.username}</strong>
    </div>
  `
}
```

That's all we need to render the online users. You can see the result in the following screenshot.

<p align="center">
  <img src="{{ site.url }}/img/posts/presence/online_users.png" />
</p>

## Typing indicator

Yet another interesting feature we can implement for our chat is to show who is typing any message at the moment.

We are going to update the presence of a user and change the `typing` value to `true` or `false` depending on the current status.

So first we need to detect that a user has been started typing and then to detect that he has been stopped typing.
We are going to use a timer with the 2 seconds timeout.

Let's start with initial values.

```js
const typingTimeout = 2000;
var typingTimer;
let userTyping = false;
```

Then inside the `if (channelRoomId)` statement, let's add following code.

```js
document.querySelector("#message-content").addEventListener('keydown', () => {
  userStartsTyping()
  clearTimeout(typingTimer);
})

document.querySelector("#message-content").addEventListener('keyup', () => {
  clearTimeout(typingTimer);
  typingTimer = setTimeout(userStopsTyping, typingTimeout);
})
```

When a user presses the key on the keyboard we are triggering "Starts Typing" event.
We are triggering "Stops Typing" event 2 seconds after a user released any key.

```js
const userStartsTyping = function() {
  if (userTyping) { return }

  userTyping = true
  channel.push('user:typing', { typing: true })
}
```

If a user just started to type, we are updating typing flag and pushing a new message to a channel, indicating that user typing right now.

```js
const userStopsTyping = function() {
  clearTimeout(typingTimer);
  userTyping = false
  channel.push('user:typing', { typing: false })
}
```

When a user finishes typing, we are updating the flag again and send the same message to indicate that user is not typing anymore.

Next we need to update `onlineUserTemplate` function to add an typing indicator next to a user's name.
```js
const onlineUserTemplate = function(user) {
  var typingIndicator = ''
  if (user.typing) {
    typingIndicator = ' <i>(typing...)</i>'
  }

  return `
    <div id="online-user-${user.user_id}">
      <strong class="text-secondary">${user.username}</strong> ${typingIndicator}
    </div>
  `
}
```

The client-side is ready, let's move on to the server-side.

We need a new callback here, to cover a "user:typing" message.

```elixir
def handle_in("user:typing", %{"typing" => typing}, socket) do
  user = Repo.get(User, socket.assigns[:current_user_id])

  {:ok, _} = Presence.update(socket, "user:#{user.id}", %{
    typing: typing,
    user_id: user.id,
    username: user.username
  })

  {:reply, :ok, socket}
end
```

We are fetching the typing flag and update user's presences.

That's it. Once a user starts to type we can see it next to his name.

<p align="center">
  <img src="{{ site.url }}/img/posts/presence/user_typing.png" />
</p>

## Wrapping up

Phoenix Presence is a powerful feature which can bring very valuable functionality to the table.
As you can see it is also pretty easy to use in Phoenix applications.

The source code of today's article you can find on [GitHub page](https://github.com/ck3g/prater/pull/10) of the project.
