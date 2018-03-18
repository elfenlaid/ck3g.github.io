---
layout: post
title: "More about Ecto and Ecto queries"
desc: "A quick glance at Ecto queries. The Pipe syntax for Ecto queries and query fragments."
keywords: "Elixir, Phoenix, Channels, WebSockets, Ecto, Ecto query, query, ecto fragments"
tags: [Elixir, Phoenix, Channels, Ecto]
excerpt_separator: <!--more-->
---

Some time ago we already mention Ecto while we were describing [Ecto models](http://whatdidilearn.info/2018/01/28/introduction-to-ecto-and-models.html#ecto).
Ecto quite a big topic, which we cannot cover in a single post.

Today I would like to talk again about Ecto and describe Ecto Queries.

<!--more-->


## Introduce Message model

Before we dig into queries, we need to extend our chat functionality with messages.
Well, our users are already able to send messages to each other, but those messages are not persisted in the database.

Let's start with the messages model.

A message is going to be related to a room and to a user. It will also contain a content of a message.
And last but not least, our message is going to be related to `Conversation` concern.

So to generate a model with all required attributes we need to run the following command.

```
→ mix phx.gen.schema Conversation.Message messages \
  room_id:references:rooms user_id:references:users content
```

Before we migrate our database let's jump into migration file and update a couple of things.

```elixir
add :content, :string, null: false
add :room_id, references(:rooms, on_delete: :delete_all)
add :user_id, references(:users, on_delete: :delete_all)
```

First, we want our `content` field to have a value all the time. So `null: false` forces that.
I've also updated `:on_delete` key to `:delete_all` in the `references` call.
That means if a related room or a user would be deleted from the database, the messages related to them would be deleted as well.
Why do we do that? Because we don't want to keep invalidated messages in our database.

Now we are ready to migrate the database.

```
→ mix ecto.migrate
```

Next thing we need to do is to update relations between our tables.

First, would be a `Conversation.Message`. In `lib/prater/conversation/message.ex` replace

```
field :room_id, :id
field :user_id, :id
```

with

```elixir
belongs_to :room, Prater.Conversation.Room
belongs_to :user, Prater.Auth.User
```

Then we need to add

```elixir
has_many :messages, Prater.Conversation.Message
```

to the `schema` block of the `Conversation.Room` and the `Auth.User` models.

That's it. The model is ready to work.


## Persisting messages

Let's create a function which would handle the creation of a message.

In the `lib/prater/conversation/conversation.ex` we need to create a new function

```elixir
alias Prater.Conversation.Message

def create_message(user, room, attrs \\ %{}) do
  user
  |> Ecto.build_assoc(:messages, room_id: room.id)
  |> Message.changeset(attrs)
  |> Repo.insert()
end
```

We need to create a message and associate it with both a user and a room.

We use a user's record and a [`build_assoc/3`](https://hexdocs.pm/ecto/Ecto.html#build_assoc/3) to build an association with a message and extend it with the `room_id`,
that makes our future message record associated with both of them.

Then we apply a changeset of the `Message` model and insert it into a database.

That part is ready, let's move to the `RoomChannel` to do the rest.


Next, in the `lib/prater_web/channels/room_channel.ex` we need to update the `handle_in` function for new messages.
We are going to store the message in the database and only then broadcast it to the subscribers.

```elixir
def handle_in("message:add", %{"message" => content}, socket) do
  room = Conversation.get_room!(socket.assigns[:room_id])
  user = find_user(socket)

  case Conversation.create_message(user, room, %{content: content}) do
    {:ok, message} ->
      message = Repo.preload(message, :user)
      message_template = %{
        content: message.content,
        user: %{username: message.user.username}
      }
      broadcast!(
        socket,
        "room:#{message.room_id}:new_message",
        message_template
      )
      {:reply, :ok, socket}

    {:error, _reason} ->
      {:reply, :error, socket}
  end
end
```

Let's try to understand what do we have here.

First, we need a room record instead of just its ID in order to pass it through to the `Conversation.create_message` function.
Then we call the function and in case of failure, we just respond with an error.
The main behavior happens when we are succeeded. We preload the user association of the message, then we form a message template and broadcast the data.

Why do we need that `Repo.preload` call? We need to get a user's username to pass back to the front-end to render a message. So we want a message to provide it to us.

To understand that let's jump to an Interactive Elixir and experiment a bit.

## Preload associations

Let's grab the first message from our database

```
iex> message = Prater.Repo.get!(Prater.Conversation.Message, 1)
%Prater.Conversation.Message{
  # ...
  content: "Hello",
  id: 1,
  user: #Ecto.Association.NotLoaded<association :user is not loaded>,
  user_id: 2
}
```

Among other attributes, we can see that the value of `user` attribute is `#Ecto.Association.NotLoaded<association :user is not loaded>`

And if we try to access the user's data we will face the following error:

```
iex> message.user.username
** (KeyError) key :username not found in:
  #Ecto.Association.NotLoaded<association :user is not loaded>
```

The thing is, that Ecto does not load all available associations for a record by default. If we want to, we need to do it explicitly.
We can use `Repo.preload` function to do that.

```elixir
iex> message_with_user = Prater.Repo.preload(message, :user)
%Prater.Conversation.Message{
  # ...
  content: "Hello",
  id: 1,
  user: %Prater.Auth.User{
    # ...
    id: 2,
    username: "user"
  },
  user_id: 2
}
```

Now we can see that `user` field contains a value of `Prater.Auth.User` model and we can access it.

```
iex> message_with_user.user.username
"user"
```

Now when we are sending messages to any room, those messages are going to be created in the database as well.

## Load messages after joining a room

We have our messages in the database, now it's time to fetch them and show on the page.

Let's start from the function for fetching messages and proceed from there.
In the `lib/prater/conversation/conversation.ex` let's create the `list_messages/2` function.


```elixir
import Ecto.Query

def list_messages(room_id, limit \\ 15) do
  Repo.all(
    from msg in Message,
    join: user in assoc(msg, :user),
    where: msg.room_id == ^room_id,
    order_by: [desc: msg.inserted_at],
    limit: ^limit,
    select: %{content: msg.content, user: %{username: user.username}}
  )
end
```

The function has two arguments: a `room_id` because we want to fetch only messages related to that room and `limit` the number of messages we are fetching.

We call a `Repo.all` with the query. If you are familiar with SQL syntax it would be easy to understand what is going on here.

First, we are stating we are going to fetch data from the `Message` model and label it as `msg`.

Then we are joining the user table. We are saying Ecto to use user association between message and user.

In the next step, we are filtering messages by a `room_id`. Notice we are using `^room_id` to keep the value unchanged in case of Pattern Matching. That is how Ecto requires us to do.

Then we are ordering messages in descending order by date of creation to grab last 15 messages.

As the last thing, we select only required information such as content and username and form it as the Map with the format we need.

All that is similar to the following SQL query.

```SQL
SELECT msg."content", user."username"
FROM "messages" AS msg
INNER JOIN "users" AS user ON user."id" = msg."user_id"
WHERE msg."room_id" = 1
ORDER BY msg."inserted_at" DESC
LIMIT 15
```

Ecto also allows us to reuse queries and then extend them. For example, we could split the "select" part from the base query in the following way.

```elixir
def list_messages(room_id, limit \\ 15) do
  query =
    from msg in Message,
    join: user in assoc(msg, :user),
    where: msg.room_id == ^room_id,
    order_by: [desc: msg.inserted_at],
    limit: ^limit

  Repo.all(
    from [msg, user] in query,
    select: %{content: msg.content, user: %{username: user.username}}
  )
end
```

For example, we can extract the query part into a separate function and use it as a predecessor for the different selects.

Now it's time to use that function. Open `lib/prater_web/channels/room_channel.ex` and update `join/3` function as follows:

```elixir
def join("room:" <> room_id, _params, socket) do
  send(self(), :after_join)

  {
    :ok,
    %{messages: Conversation.list_messages(room_id)},
    assign(socket, :room_id, room_id)
  }
end
```

Now when some user joins a room, we are providing the list of recent messages back.
All we need to do now is the get those messages on the client-side and render them.

In the `assets/js/socket.js` let's change the following line

```js
.receive("ok", resp => { console.log("Joined successfully", resp) })
```

to

```js
.receive("ok", resp => {
  console.log("Joined successfully", resp)
  resp.messages.reverse().map(message => renderMessage(message))
})
```

As soon as our `renderMessage` adds messages to the end, we need to flip array and then map through it in order to render messages in the correct order.

That is pretty much it. Now if we join any room, we would see messages which were created earlier.


Let's take a look at a couple of more features Ecto gives us.

## Pipe Syntax

Ecto also supports the Pipe Syntax for queries.

The shorter version of our previous query would look like:

```
iex> import Ecto.Query
iex> room_id = 1
iex> Prater.Conversation.Message |>
...> select([msg], %{id: msg.id, content: msg.content}) |>
...> where([msg], msg.room_id == ^room_id) |>
...> Prater.Repo.all

[
  %{content: "Hello again", id: 4},
  %{content: "Hey ", id: 3},
  %{content: "Hello there", id: 2}
]
```

Although I didn't manage to find how to properly use "joins" for that syntax.
If you know how to do that, let me know in the comments below.

Which type of syntax to use, it is completely up to you.

## Fragments

Ecto provides a lot of functionality to work with databases. Although it does not cover 100% functionality some databases have.

When you are facing a situation in which you need to use some features of a database which is not supported by Ecto (yet).
You can use a feature called "query fragments". You can construct a small piece of SQL and Ecto will safely pass it down to the database level.


We can see how to use fragments in the following example and the SQL it produces.

```
iex> import Ecto.Query
iex> Prater.Repo.all(
      from r in Prater.Conversation.Room,
      where: fragment("lower(name) = ?", ^String.downcase("Lobby")))
```

```sql
SELECT r0."id", r0."description", r0."name"
FROM "rooms" AS r0
WHERE (lower(name) = $1) ["lobby"]
```

```elixir
[
  %Prater.Conversation.Room{
    description: "The general chat room. Everybody welcome here.",
    id: 1,
    name: "Lobby"
  }
]
```

## Wrapping up

Today we have we have covered an important piece of Ecto functionality. Ecto queries.
We haven't covered every single opportunity Ecto gives us, but that wasn't the goal of that article.
I think that can give you a starting point to learn about Ecto abilities. Some of them I think we will cover someday in the following articles.
Stay tuned.

The related code you can find on a [GitHub page](https://github.com/ck3g/prater/pull/11).
