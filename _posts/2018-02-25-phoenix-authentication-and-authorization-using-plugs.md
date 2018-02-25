---
layout: post
title: "Phoenix: Authentication and Authorization using Plugs"
desc: "We are going to introduce you into Phoenix Plugs and how to apply that knowledge to build an Authorization functionality in your application."
keywords: "Elixir, Phoenix, Auth, Authentication, Authorization, Plugs, Connection"
tags: [Elixir, Phoenix]
excerpt_separator: <!--more-->
---

In [the previous article](http://whatdidilearn.info/2018/02/18/authentication-in-phoenix.html), we have implemented authentication solution.
But we still have some spots to cover.
Our users still can access every page even though they are not logged in.

Today we will introduce some improvements in our authentication solution and do it by using the Plugs.

Let's get started.

<!--more-->

## What is a Plug?

Plugs are some sort of layers in a Phoenix application. Which can be injected in between initial request and final response.
They are small reusable prices and can be used to transform the connection.

Every Phoenix request starts with a connection and then goes deep down by slightly changing it on its way.

So Plug is some piece of code which receives a connection, changes it slightly and returns back.

## Types of the Plugs: module and function

There are two types of plugs in the Phoenix. They are module plugs and function plugs. We are going to cover both of them in this article.

The structure of the Module plug is pretty simple. It is a module with two functions is in it:

* 'init/1' - prepares the arguments (if needed) to be passed to `call/2`
* `call/2` - accepts the connection, transforms it and returns it back

It would be easier to understand when we will implement them.

## Set current user

At the moment in our application, we are keeping the `current_user_id` in the session and only access it through `Prater.Auth.current_user/1` functions.
The connection itself does not know about a current user.

Let's implement a functionality, that on every page request we are going to get the user ID from the session and assign the user itself to the connection.

Let's create a new file `lib/prater_web/plugs/set_current_user.ex` for our future plug.

There is no strict rule where to keep plugs in your application, but it seems to me, keeping them in the same directory with controllers makes more sense.
Thus I'm going to keep plugs in the `lib/prater_web/plugs` directory.

And the implementation of the plug looks like:

```elixir
defmodule PraterWeb.Plugs.SetCurrentUser do
  import Plug.Conn

  alias Prater.Repo
  alias Prater.Auth.User

  def init(_params) do
  end

  def call(conn, _params) do
    user_id = Plug.Conn.get_session(conn, :current_user_id)

    cond do
      current_user = user_id && Repo.get(User, user_id) ->
        conn
        |> assign(:current_user, current_user)
        |> assign(:user_signed_in?, true)
      true ->
        conn
        |> assign(:current_user, nil)
        |> assign(:user_signed_in?, false)
    end
  end
end
```

So that is a module plug and in order to behave like a plug, we need to import `Plug.Conn` module to extend its functionality.
Then we have `init/1` function, which in our case does nothing.

The rest of the stuff happens in the `call/2`. The first thing we do is we retrieve the user ID from the session.
Then in the `cond` construction, we are trying to fetch the user with that ID from the DataBase.
If we succeeded we are assigning the user map to the `:current_user` of the connection and we also assign `:user_signed_in?` to `true`, because we know that the user is signed in. If we are not succeeded we assign a `nil` to `:current_user` and `false` to `:user_signed_in?`.

That is pretty much it that does the plug do. If the user signed in, we are updating the connection so it would know about a user. And we set it to nil otherwise.

Now we need to start using the plug somehow.

Before we do that we can ask ourselves a question: "When do we need that functionality to be triggered?". You ask the same question for every plug you are trying to implement. The answer helps to figure out next steps.

In our case, we want this plug to be used everywhere in the browser. So the `Router` would be the right place to plug it.

In the `lib/prater_web/router.ex` you may see the pipelines already and even the other plugs which come with the Phoenix.
We want our Plug to work in the browser (we don't care about API for now). That means we need to inject it into `pipeline :browser` as follows.

```elixir
pipeline :browser do
  # ...
  plug PraterWeb.Plugs.SetCurrentUser
end
```

That's it. The plug has been plugged in and works.

To see that in action we need to replace the following condition in the layout file (`lib/prater_web/templates/layout/app.html.eex`):

```erb
<%= if Prater.Auth.user_signed_in?(@conn) do %>
  <nav class="my-2 my-md-0 mr-md-3">
    Signed in as: <strong><%= Prater.Auth.current_user(@conn).username %></strong>
```

with the:

```erb
<%= if @user_signed_in? do %>
  <nav class="my-2 my-md-0 mr-md-3">
    Signed in as: <strong><%= @current_user.username %></strong>
```

We can access the assigns of the connection in several ways:

* We can access it directly by using `@` sign in the views. As we did with the `@user_signed_in?`
* We can access by calling `conn.assigns[:user_signed_in?]`
* Or we can access by calling `conn.assigns.user_signed_in?`

You decide which approach to use depending on your situation.

We can also remove the following functions, as soon as we are not using them anymore.

```
Prater.Auth.current_user/1
Prater.Auth.user_signed_in?/1
```

Now if you run the page you can see that the sign in and sign out buttons are rendered properly. That means our plug works.

## Authenticate user

Let's make a quick look at our application, we have the authentication functionality in place,
but our guests can access every page without being signed in. That is not how we want it to be.

Let's restrict guests from access rooms' pages except for the index page.

As a first solution, we can implement a function plug to do the job.

On top of our `RoomController` let's start with plugging the plug.

```elixir
plug :authenticate_user when action in [:new, :create, :show, :edit, :update, :delete]
```

Here we are describing that we would like to use the `:authenticate_user` plus as a function.
Also, we want it to be activated only for the following actions.

Let's jump to the implementation. Define the private function right inside that controller.

```elixir
defp authenticate_user(conn, _params) do
  if conn.assigns.user_signed_in? do
    conn
  else
    conn
    |> put_flash(:error, "You need to sign in or sign up before continuing.")
    |> redirect(to: session_path(conn, :new))
    |> halt()
  end
end
```

What do we do here? We checked if the user signed in and if he is, we just the return the connection back without doing anything else.

If we have a guest visiting a page, then we set a flash message, redirect him to a sign in page and also [halting](https://hexdocs.pm/plug/Plug.Conn.html#halt/1) the connection in order to prevent rest of the plugs to be invoked.

If you try to access a new room page now being signed out you should see the work of the plug in action.

A tiny piece before we move on.

You may notice that we have provided a long list of actions to use our plug.

```elixir
plug :authenticate_user when action in [:new, :create, :show, :edit, :update, :delete]
```

In fact, those are all available actions except `index`. Can we somehow make the list shorter?
Yes, we can.

We can describe which actions to avoid instead by:

```
plug :authenticate_user when action not in [:index]
```

Ok. The functionality works fine. But we are probably want to use the same functionality in other controllers. We don't have them for now, but anyway, you will probably have them in another application.

We don't want to implement the same function plug in every controller. So for that case, the module plug would work better. Let's refactor that.

## Extract authenticate user into Module Plug

As soon as it would be a module we need a new file for that: `lib/prater_web/plugs/authenticate_user.ex` with the following content:

```elixir
defmodule PraterWeb.Plugs.AuthenticateUser do
  import Plug.Conn
  import Phoenix.Controller

  alias PraterWeb.Router.Helpers

  def init(_params) do
  end

  def call(conn, _params) do
    if conn.assigns.user_signed_in? do
      conn
    else
      conn
      |> put_flash(:error, "You need to sign in or sign up before continuing.")
      |> redirect(to: Helpers.session_path(conn, :new))
      |> halt()
    end
  end
end
```

All in all that is the same function we have in the `RoomController` with small changes.
On top, if importing the `Plug.Conn` module we also need to import the `Phoenix.Controller` module, because we have some stuff related to controllers functionality.
We also need to use `Router.Helpers` module in order to call `session_path` function.

That is it. Nothing unusual.

The last piece we need to change to glue all together is the way how we are plugging that plug.

We need to change the following line

```elixir
plug :authenticate_user when action not in [:index]
```

to use module name instead

```elixir
plug PraterWeb.Plugs.AuthenticateUser when action not in [:index]
```

Ah yes, and we can remove `RoomController.authenticate_user` function. We don't need it anymore.

That's it. It works and we can reuse it for different controllers.

## Authorize user

Now we have our pages somehow secured from the guests. The guests simply cannot access them.
But signed in users can manage every room in the app. That means they can change and remove rooms event though those rooms do not belong to them.

Let's go further and allow users to manage rooms they created.

As usual, there are some prerequisites to implement that task. At the moment we are not keeping the track of which room belongs to which user.
We need to link them together before we can proceed.

### Define association between users and rooms

To link rooms and user's together we need to update our database and keep the track of `user_id` in the `rooms` table.
Because we want a room to belong to a user. Also, a user can create several rooms.

For every change we do in the database we need to do those using migrations.

```elixir
→ mix ecto.gen.migration add_user_id_to_rooms
* creating priv/repo/migrations
* creating priv/repo/migrations/20180225104638_add_user_id_to_rooms.exs
```

and open the created file `*_add_user_id_to_rooms.exs` (for you it will have a different timestamp in the file name) and update the `change` function.

```elixir
defmodule Prater.Repo.Migrations.AddUserIdToRooms do
  use Ecto.Migration

  def change do
    alter table(:rooms) do
      add :user_id, references(:users)
    end
  end
end
```

We want to alter the table rooms and define `user_id` column to be the reference to a `users` table.

We can execute the migration afterward.

```
→ mix ecto.migrate
[info] == Running Prater.Repo.Migrations.AddUserIdToRooms.change/0 forward
[info] alter table rooms
[info] == Migrated in 0.0s
```

We have `user_id` column now, but our models still do not aware about that relation.

In the `Room` model we need to update the `schema` and explicitly say that the room belongs to a user, and a user is being represented by `Prater.Auth.User` model.

```elixir
schema "rooms" do
  # ...
  belongs_to :user, Prater.Auth.User
  # ...
end
```

The similar to the `User` model. A user can own many rooms. A room is represented by `Prater.Conversation.Room` model.

```elixir
schema "users" do
  # ...
  has_many :rooms, Prater.Conversation.Room
  # ...
end

```

Now models know about associations between them. The next thing is we need to associate the room we are going to create with the current user.



When we create our room in the `RoomController.create` action we are hiding all the logic inside `Conversation.create_room` function.
That function does not know anything about the current user. So we need to update the function definition and pass the user inside.

We need to change the way how we call it.
```elixir
case Conversation.create_room(room_params) do
```

to

```elixir
case Conversation.create_room(conn.assigns.current_user, room_params) do
```

Then we need to update the function itself and associate a room with a current user.
We can achieve that using a [Ecto.build_assoc/2](https://hexdocs.pm/ecto/Ecto.html#build_assoc/3) function.

That is how our function should look like before and after.

```elixir
# Before
def create_room(attrs \\ %{}) do
  %Room{}
  |> Room.changeset(attrs)
  |> Repo.insert()
end

# After
def create_room(user, attrs \\ %{}) do
  user
  |> Ecto.build_assoc(:rooms)
  |> Room.changeset(attrs)
  |> Repo.insert()
end
```

Instead of empty `%Room{}` map, first, we are building an association and then do the rest by passing it into `changeset`.

That is it. Go and create a room.

Once you've done it you can try to fetch the room from the `iex` session:

```
iex> Prater.Repo.get(Prater.Conversation.Room, 11)
[debug] QUERY OK source="rooms" db=1.2ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."user_id", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 WHERE (r0."id" = $1) [11]
%Prater.Conversation.Room{
  __meta__: #Ecto.Schema.Metadata<:loaded, "rooms">,
  description: nil,
  id: 11,
  inserted_at: ~N[2018-02-25 11:42:53.589542],
  name: "User's room",
  topic: nil,
  updated_at: ~N[2018-02-25 11:42:53.593703],
  user: #Ecto.Association.NotLoaded<association :user is not loaded>,
  user_id: 10
}
```

We can see now how it associated with the user `user_id: 10` and has a `user` association as well.

That's it. Now we can proceed with the Authorization Plug.

In that case, we would like to check if a user can or cannot access the rooms.
So the function plug would be a valid option to use, we are not going to reuse it, other controllers.

Let's start with enabling the plug. We want it to work only for the `edit`, `update`, and `delete` actions.

```elixir
plug :authorize_user when action in [:edit, :update, :delete]
```

And the implementation itself.

```elixir
defp authorize_user(conn, _params) do
  %{params: %{"id" => room_id}} = conn
  room = Conversation.get_room!(room_id)

  if conn.assigns.current_user.id == room.user_id do
    conn
  else
    conn
    |> put_flash(:error, "You are not authorized to access that page")
    |> redirect(to: room_path(conn, :index))
    |> halt()
  end
end
```

Here the `_params` are not the same as the params we get in the actions, so we need to fetch Room ID manually.
If we working with one of the `edit`, `update`, or `delete` actions we have that in URL param: `/rooms/:id`.
All we need is to [Pattern Match](http://whatdidilearn.info/2017/09/21/pattern-matching-in-elixir.html) it from our connection.

```elixir
%{params: %{"id" => room_id}} = conn
```

Then we need to find the room

```elixir
room = Conversation.get_room!(room_id)
```

and then check if current user ID matches with the `user_id` field of the Room.

Then we check if a user is the owner of a room, then he can proceed. Others will the flash messages and will be redirected to the rooms index page.

That is all we need to make that plug work.

Yes, in that example we are fetching the room from the DataBase twice.
Which we usually should avoid. But let's leave it for now as it is.

Or you can have a homework and implement yet another plug function to fetch the room and assign it to the connection.

### Small improvements

Most likely we would like to check the same "can access" (`conn.assigns.current_user.id == room.user_id`) condition in other places.
For example to hide "Edit" and "Delete" buttons from the page or even have more improvement logic to check if a user can manage it or not.

Let's extract it into a separate module and keep it in `lib/prater/auth/authorizer.ex`:

```elixir
defmodule Prater.Auth.Authorizer do
  def can_manage?(user, room) do
    user && user.id == room.user_id
  end
end
```
Now we replace

```elixir
if conn.assigns.current_user.id == room.user_id do
```

with

```elixir
if Authorizer.can_manage?(conn.assigns.current_user, room) do
```

and describe an alias on top of the controller

```elixir
alias Prater.Auth.Authorizer
```

Now we can hide the "Edit" and "Delete" buttons if a current user cannot manage a room:

Open the `lib/prater_web/templates/room/show.html.eex` file and wrap buttons with the condition:

```erb
<%= if Prater.Auth.Authorizer.can_manage?(@current_user, @room) do %>
  <div>
    <%= link "Edit", to: room_path(@conn, :edit, @room.id), class: "btn btn-default" %>
    <%= link "Delete", to: room_path(@conn, :delete, @room), method: :delete, data: [confirm: "Are you sure?"], class: "btn btn-danger" %>
  </div>
<% end %>
```

That's it. Now the manage buttons can see only an owner of a room.


## Wrapping up

This time we have learned about plugs in Phoenix. Which might be scary from the beginning, but pretty easy to use.
We know there are two types of plugs such as module plug and function plug.
We have also learned how to use plugs inside the router and inside controllers.

The source code of the current implementation you can find on [GitHub](https://github.com/ck3g/prater/pull/7) page.

See you next time.
