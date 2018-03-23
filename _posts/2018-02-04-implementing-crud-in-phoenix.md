---
layout: post
title: "Implementing CRUD in Phoenix"
desc: "Learn Phoenix by implementing CRUD functionality. Learn how to create, read, update and delete Phoenix resources."
keywords: "Elixir, Phoenix, CRUD"
tags: [Elixir, Phoenix]
---

Last time we have covered the models and we have created a rooms model to keep information about them.
Even though those rooms are in the database now, we are not able to manage them.
Let's do that by implementing all available CRUD operations.

## What is CRUD?

CRUD is basically an acronym for the Create, Read, Update, and Delete.
Which are basic functions of persistence storage, but also mostly used in building web interfaces.

We are going to implement them one by one.

On the left side of our page, we already have the list of the rooms.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_crud/before_crud.png" />
</p>

Let's implement functionality to create them.



## C for Create

How would we create the room in our app?

As a first step, a user will probably click the "New Room" button.
Then sees the form with fields he needs to fill in. He fills that form in and clicks "Submit" button.
Using submitted data we are going to create the record in the database and redirect the user back to the list.

So we need a button. Let's render it next to our "Rooms" title.

Replace the following line in the `lib/prater_web/templates/room/index.html.eex` file:

```html
<h3>Rooms</h3>
```

with

```html
<h3>Rooms <%= link "+", to: "/rooms/new", class: "btn btn-success" %></h3>
```

Here we have used [Phoenix HTML link](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Link.html#link/2) helper to render a link on the page. That is equivalent of the following code:

```html
<h3>Rooms <a href="/rooms/new", class: "btn btn-success">+</a></h3>
```

So now we have a link next to our "Rooms" title. We click it, we get an error message:

```
no route found for GET /rooms/new (PraterWeb.Router)
```

The Phoenix does not aware of that URL and does not know which action should be used.

To fix that we need to update Router in the `lib/prater_web/router.ex` file.
We already have a route for `RoomController`.
Let's add new route right next to it.

```elixir
get "/", RoomController, :index # <- This line already exists
get "/rooms/new", RoomController, :new
```

Then we follow the error messages and fix them. To fix:

```
function PraterWeb.RoomController.new/2 is undefined or private
```

We create `new` action in our `RoomController`:

```elixir
def new(conn, _params) do
  render(conn, "new.html")
end
```

As we already know from the [Controllers and Views](http://whatdidilearn.info/2018/01/21/controllers-views-and-templates.html) article, an action has to either render the view or redirect user somewhere.
We need to render the form with inputs for the user, so we are rendering a new page.
We also need to create a `lib/prater_web/templates/room/new.html.eex` template in order to avoid the following error:

```
Could not render "new.html" for PraterWeb.RoomView
```

We are going to create it with the simple title.

```html
<h2>Add new room</h2>
```

We do it to check we are on the right page. And we are

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_crud/new_title.png" />
</p>

Now let's render the form by using the following markup.

```html
<h2>Add new room</h2>

<%= form_for @changeset, room_path(@conn, :create), fn f -> %>
  <div class="form-group">
    <%= label f, :name, class: "control-label" %>
    <%= text_input f, :name, class: "form-control" %>
    <%= error_tag f, :name %>
  </div>

  <div class="form-group">
    <%= label f, :description, class: "control-label" %>
    <%= textarea f, :description, class: "form-control", rows: 5 %>
    <%= error_tag f, :description %>
  </div>

  <div class="form-group">
    <%= submit "Submit", class: "btn btn-primary" %>
  </div>
<% end %>
```

Now we see the following error on the page:

```
assign @changeset not available in eex template.
```

That is correct. We didn't pass the `@changeset` down to the template yet.

```elixir
def new(conn, _params) do
  alias Prater.Conversation.Room
  changeset = Room.changeset(%Room{}, %{})
  render conn, "new.html", changeset: changeset
end
```

We set it to an empty `Room` and pass it to `render` function. Now the error message has been changed.

```
No function clause for PraterWeb.Router.Helpers.room_path/2 and action :create.
```

We are using `room_path(@conn, :create)` following function as an target action for our form.
But we didn't define it yet. To do that we need to create a new route.

Phoenix uses [RESTful way](https://en.wikipedia.org/wiki/Representational_state_transfer#Applied_to_Web_services) of naming urls.
We need `create` action. The RESTful way expects `create` action to have [POST HTTP method](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_methods).

```elixir
post "/rooms", RoomController, :create
```

Now it works, we can see the form on the page

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_crud/new_form.png" />
</p>

As a next step, we need to submit our form. We didn't implement functionality for that. So we are getting the following error:

```
function PraterWeb.RoomController.create/2 is undefined or private
```

We need to define our `create` action. Let's paste the complete implementation and figure out what does it do in small steps:

```elixir
def create(conn, %{"room" => room_params}) do
  alias Prater.Conversation.Room
  alias Prater.Repo
  %Room{}
  |> Room.changeset(room_params)
  |> Repo.insert()
  |> case do
    {:ok, room} ->
      conn
      |> put_flash(:info, "Room created successfully.")
      |> redirect(to: room_path(conn, :index))

    {:error, %Ecto.Changeset{} = changeset} ->
      render(conn, "new.html", changeset: changeset)
  end
end
```

We have a couple of [aliases](http://whatdidilearn.info/2017/10/16/modules-in-elixir.html#alias) to reduce typing.
Then we initialize the room and apply our `Room.changeset` using parameters from the form.
Then we try to save it into the database.
If it succeeded and `Repo.insert` returns us `{:ok, room}` then we are going to redirect the user back to the home page and display a message to him.
If the insert was failed we will render the form where the user will be able to see validation errors.

If we try to create a new room using our form, it should work now.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_crud/room_created.png" />
</p>

The functionality works. Before we move to the next functionality of CRUD, let's stop for a while and do some improvements.

### Small refactoring

Let's hide some implementation details into `Conversation` context. We are going to extract it out of `new` action.

```elixir
def new(conn, _params) do
  changeset = Conversation.change_room(%Room{})
  render conn, "new.html", changeset: changeset
end
```

We are also moved the following [aliases](http://whatdidilearn.info/2017/10/16/modules-in-elixir.html#alias) to the top of the controller module.

```elixir
alias Prater.Conversation
alias Prater.Conversation.Room
```

The function `Conversation.change_room/1` does not exist yet, so we need to create it. Open `lib/prater/conversation/conversation.ex` file and add following implementation.

```elixir
def change_room(%Room{} = room) do
  Room.changeset(room, %{})
end
```

Now let's move to `create` function. Let's extract the creation logic into `Conversation.create_room/1` function.
Our create function should look like:

```elixir
def create(conn, %{"room" => room_params}) do
  case Conversation.create_room(room_params) do
    {:ok, room} ->
      conn
      |> put_flash(:info, "Room created successfully.")
      |> redirect(to: room_path(conn, :index))

    {:error, %Ecto.Changeset{} = changeset} ->
      render(conn, "new.html", changeset: changeset)
  end
end
```

The same procedure as we did with `new` action, we need to create `Conversation.create_room/1` function.

```elixir
def create_room(attrs \\ %{}) do
  %Room{}
  |> Room.changeset(attrs)
  |> Repo.insert()
end
```

Why did we do all these steps? Now our controller looks more compact and some implementation details are hidden in the `Conversation` context.

Now you can go and create one more room to check if our code still working. It should work.
Now we are moving to the next step.

## R for Read

The next part of the CRUD is Read. We are able to see a list of the rooms. So this piece partially covered. The missing part is to show the page of a particular room.

The `show` action stands for that.

We start by adding the link.

Open the `lib/prater_web/templates/room/index.html.eex` file and change following line:

```html
<li class="list-group-item"><%= room.name %></li>
```

to

```html
<li class="list-group-item">
  <%= link room.name, to: room_path(@conn, :show, room.id) %>
</li>
```

Now we are going to have the link to the room page instead of text.

We are still missing the route. Let's create it.

```elixir
get "/rooms/:id", RoomController, :show
```

By specifying `:id` in the URL, we are saying that our URL will have `id` parameter, which can use to capture the room's ID.

We have a route now, we need an action:

```elxir
def show(conn, _params) do
  render conn, "show.html"
end
```

That is not the first action we are creating. We already have some experience doing that. We know we need a template file  `lib/prater_web/templates/room/show.html.eex`. Let's use a simple title to check if what we did before is working.

```html
<h2>Room details</h2>
```

It works. Cool. But we cant see the room details, let's fix that.
We already passing the room ID into the URL. Let's grab it and fetch the room information.
Once we do that we would need to pass it down to the template.

```elixir
def show(conn, %{"id" => id}) do
  room = Prater.Repo.get!(Room, id)
  render(conn, "show.html", room: room)
end
```

Now we have the room's data and can display it on the page. Let's update our template.

```html
<h2><%= @room.name %></h2>

<div><%= @room.description %></div>
```

And we can see room's details

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_crud/show_lobby.png" />
</p>

### Small refactoring

As usual, once we have done the implementation we can slow down for a while and refactor our code.
Let's hide implementation details in the `Conversation` module, by extracting:

```elixir
Prater.Repo.get!(Room, id)
```

into new `get_room!/1` function

```elixir
def get_room!(id), do: Repo.get!(Room, id)
```

so the `show` action would look like:

```elixir
def show(conn, %{"id" => id}) do
  room = Conversation.get_room!(id)
  render(conn, "show.html", room: room)
end
```

That's it for "Read".

## U for Update

We are going to implement the update functionality.
Similar to "create" we need a link, so a user will be able to navigate to the page with the edit form.


```html
<div>
  <%= link "Edit", to: room_path(@conn, :edit, @room.id), class: "btn btn-default" %>
</div>
```

We need a route.

```elixir
get "/rooms/:id/edit", RoomController, :edit
```
The same as with `show` URL, we need a room's ID to know which room we are going to update.
Now let's create the `edit` action which will be responsible to render the form.

```elixir
def edit(conn, %{"id" => id}) do
  room = Conversation.get_room!(id)
  changeset = Conversation.change_room(room)
  render(conn, "edit.html", room: room, changeset: changeset)
end
```

With some small difference comparing to show, we need to pass changeset to the template in order to render the form.
Speaking of which. There it is:

```html
<h2>Edit room: <%= @room.name %></h2>

<%= form_for @changeset, room_path(@conn, :update, @room), fn f -> %>
  <div class="form-group">
    <%= label f, :name, class: "control-label" %>
    <%= text_input f, :name, class: "form-control" %>
    <%= error_tag f, :name %>
  </div>

  <div class="form-group">
    <%= label f, :description, class: "control-label" %>
    <%= textarea f, :description, class: "form-control", rows: 5 %>
  </div>

  <div class="form-group">
    <%= submit "Submit", class: "btn btn-primary" %>
  </div>
<% end %>
```

And we also need the last route to make it work.

```elixir
put "/rooms/:id", RoomController, :update
```

Unlike `create` route, we need [PUT HTTP method](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_methods) for updates.

The edit page has been loaded with the form on it and the "name" and "description" fields contain the room's data as well.
Try to change something and submit the form. You will see following error:

```
function PraterWeb.RoomController.update/2 is undefined or private
```

Of course. Let's create the function.

```elixir
def update(conn, %{"id" => id, "room" => room_params}) do
  room = Conversation.get_room!(id)

  room
  |> Room.changeset(room_params)
  |> Prater.Repo.update()
  |> case do
    {:ok, room} ->
      conn
      |> put_flash(:info, "Room updated successfully.")
      |> redirect(to: room_path(conn, :show, room))
    {:error, %Ecto.Changeset{} = changeset} ->
      render(conn, "edit.html", room: room, changeset: changeset)
  end
end
```

First, we found the room we need to update.
Then we are trying to apply `Room.changeset` with the parameters passed from the form.
If we succeed with an update and get `{:ok, room}` back, we are redirecting a user back to room's page and display a "Room update successfully" message.
Otherwise, we render the form with validation messages on it.

Check it. It should work.

### Small refactoring

Refactoring is becoming a new tradition and it should be. After a bunch of changes, we need to look around and improve some parts of the code.

As usually let's hide the implementation details in `Conversation` context and replace following lines in `update` action:

```elixir
room
|> Room.changeset(room_params)
|> Prater.Repo.update()
|> case do
```

with

```elixir
case Conversation.update_room(room, room_params) do
```

and of course, implement that function

```elixir
def update_room(%Room{} = room, attrs) do
  room
  |> Room.changeset(attrs)
  |> Repo.update()
end
```

You have probably notice that we have implemented two forms for the create and edit purposes.
But the forms barely have a difference. The only difference is the URL we are using to submit the form. We can also eliminate code duplication.

Let's extract the form in the `lib/prater_web/templates/room/form.html.eex` file with the following content.

```html
<%= form_for @changeset, @action, fn f -> %>
  <div class="form-group">
    <%= label f, :name, class: "control-label" %>
    <%= text_input f, :name, class: "form-control" %>
    <%= error_tag f, :name %>
  </div>

  <div class="form-group">
    <%= label f, :description, class: "control-label" %>
    <%= textarea f, :description, class: "form-control", rows: 5 %>
  </div>

  <div class="form-group">
    <%= submit "Submit", class: "btn btn-primary" %>
  </div>
<% end %>
```

That is the same form we have in our templates with the small difference. It has an `@action` variable instead of target URL.

Now we can update a `new.html.eex` template to use that form:

```html
<h2>Add new room</h2>

<%= render "form.html", Map.put(assigns, :action, room_path(@conn, :create)) %>
```

We are injecting another template inside existing one and extend `assigns` map with the new `action` key which holds the target URL for the form.


We are also updating the `edit.html.eex` template with similar `render` call:

```html
<%= render "form.html", Map.put(assigns, :action, room_path(@conn, :update, @room)) %>
```

The only URL is differs comparing to `new.html.eex`.

We are done. Check everything again and be sure it is still working. Let's move on.

## D for Delete

To implement the delete functionality we will start with the link again.

```html
<%= link "Delete", to: room_path(@conn, :delete, @room), method: :delete, data: [confirm: "Are you sure?"], class: "btn btn-danger" %>
```

This link a little bit longer than usual. Now we have `method: :delete` parameter. That is because the `link` helper uses GET HTTP method by default.
In case of delete, we need the method to be DELETE.
We also want to prevent accidental removal of any room. So we ask the user to confirm his choice by passing `data: [confirm: "Are you sure?"]`.

We need a route for that:

```elixir
delete "/rooms/:id", RoomController, :delete
```

And the route has DELETE HTTP method.

The first implementation of the delete action looks like:

```elixir
def delete(conn, %{"id" => id}) do
  room = Conversation.get_room!(id)
  {:ok, _room} = Prater.Repo.delete(room)

  conn
  |> put_flash(:info, "Room deleted successfully.")
  |> redirect(to: room_path(conn, :index))
end
```

We are fetching the room by its ID. And we call `Repo.delete` to delete it from the database.
Then we redirect a user back to the rooms list.

Check it. Is it working?

That's it. This time we don't need any views. Because all we do we are removing the room and redirecting a user.

## Small refactoring

Let's extract the `delete` call.

```elixir
{:ok, _room} = Prater.Repo.delete(room)
```
into

```elixir
{:ok, _room} = Conversation.delete_room(room)
```

with the following implementation

```elixir
def delete_room(%Room{} = room) do
  Repo.delete(room)
end
```

Done.

One more thing.

If we look into our Router. We can see a bunch of routes related to rooms. They all have similar goals.
Is it possible to refactor? Yes, it is.

Meet the `resources`.

As soon as we follow Phoenix convention to define our routes. All of the following routes

```elixir
get "/rooms/new", RoomController, :new
post "/rooms", RoomController, :create
get "/rooms/:id", RoomController, :show
get "/rooms/:id/edit", RoomController, :edit
put "/rooms/:id", RoomController, :update
delete "/rooms/:id", RoomController, :delete
```

can be replaced with a one-liner

```elixir
resources "/rooms", RoomController
```

The `resources` function will create all required routes for us.

We can check it by running `phx.routes` task:

```
→ mix phx.routes
room_path  GET     /                PraterWeb.RoomController :index
room_path  GET     /rooms           PraterWeb.RoomController :index
room_path  GET     /rooms/:id/edit  PraterWeb.RoomController :edit
room_path  GET     /rooms/new       PraterWeb.RoomController :new
room_path  GET     /rooms/:id       PraterWeb.RoomController :show
room_path  POST    /rooms           PraterWeb.RoomController :create
room_path  PATCH   /rooms/:id       PraterWeb.RoomController :update
           PUT     /rooms/:id       PraterWeb.RoomController :update
room_path  DELETE  /rooms/:id       PraterWeb.RoomController :delete
```

As you can see we are not missing any route. All of them are here.

So that concludes the "D".

## Wrapping up

To wrap it up. Today we have learned how to create manageable resources using CRUD actions. We have covered all the standard CRUD functions.
Now out users can create a room, can see the room's details, they can also update the room and even delete it.

You can find all the changes we have done here on the [GitHub page](https://github.com/ck3g/prater/pull/3) of the project.

You may say, that it is possible to create all these CRUD actions using the `mix phx.gen.html` generator. And it would be much faster.

You are right. But!

It is always worth to know how every piece works and be able to implement it manually. Also, you don't need all CRUD actions for every resource all the time. Sometimes, or even, most of the time, you would need only several of them.

In the end, it is up to you which approach to use. Because now you know every step.

Take care. See you next time.
