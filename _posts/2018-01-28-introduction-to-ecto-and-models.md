---
layout: post
title: "Introduction to Ecto and Models"
desc: "Here we are introducing you to Ecto library. We continue to talk about MVC and especially models."
keywords: "Elixir, Phoenix, Model, Ecto, Changeset, Repo, DataBase"
tags: [Elixir, Phoenix]
---

In the previous article, we have learned about MVC and covered a little bit more how to work with Controllers and Views.
One piece is still missing. The models. Let's talk about them now.

As we remember the models are responsible for managing the data of the application.
We also remember that models should hide the implementation details.

In our last task, we have provided the basic page layout and displayed the static list of the rooms on the page.
Now it would be nice to get that list from somewhere else.

## Rooms list

We remember that it does not matter for a controller and a view where is the data comes from. Thus let's create a function to provide the list of rooms.

Sometimes it is easier to implement a functionality from inside out. We already have a page with the list of rooms in the `lib/prater_web/templates/room/index.html.eex`.

Let's change the following line in there:

```html
<li class="list-group-item">Lobby</li>
```

to

```html
<%= inspect @rooms %>
```

Here we have embedded Elixir expression into HTML template using [Embedded Elixir](https://hexdocs.pm/eex/EEx.html). The `<%= "value" %>` construct means that the result of the expression would be present inside an HTML template.

Currently, we will just `inspect` the result and then update it soon.

Now the page welcomes us with the following error:

```
assign @rooms not available in eex template.
```

That is correct. We are using a variable which our template does not know about. Now we need to assign the value to it in the controller and pass down to the template.

Let's change our `RoomController.index` function to looks in the following way:

```elixir
def index(conn, _params) do
  rooms = Prater.Conversation.list_rooms()
  render conn, "index.html", rooms: rooms
end
```

Here we are getting a list of the rooms and pass it to the view (and then to the template).
The error has been changed to

```
function Prater.Conversation.list_rooms/0 is undefined (module Prater.Conversation is not available)
```

That is also correct. We don't have that module yet. So let's create it in the `lib/prater/conversation/conversation.ex` file.

```elixir
defmodule Prater.Conversation do
  def list_rooms do
    [
      %{name: "Lobby", description: "The general chat room. Everybody welcome here."}
    ]
  end
end
```

Here we have a module with the function we need. The function returns a list of [maps](http://whatdidilearn.info/2017/09/27/a-quick-glance-on-basic-types-in-elixir.html#maps) with the rooms' details.

Now if we render our page we will see that list under the "Rooms" section.

<p align="center">
  <img src="{{ site.url }}/img/posts/ecto_and_models/rooms_inspect.png" />
</p>

OK. So it's time to update the ~~view~~ template to properly display that list. How do we do that?
We need to iterate through every room item in the list, grab the name of it, and render as a `<li>` element.

In our template file, we need to replace

```html
<%= inspect @rooms %>
```

with

```html
<%= for room <- @rooms do %>
  <li class="list-group-item"><%= room.name %></li>
<% end %>
```

We have used a [comprehension](http://whatdidilearn.info/2017/11/12/control-flow-in-elixir.html#comprehensions) here to iterate through a list of the rooms and render the `<li>` element for each room.

And now our page looks as it was at the beginning. We see the rooms list again.

<p align="center">
  <img src="{{ site.url }}/img/posts/ecto_and_models/rooms_list.png" />
</p>

Let's quickly sum up what we did till now. We have created a model which provides us data about rooms.
Our controller and view use that data to render a list of the rooms on the page.
Yes, that data is hardcoded at the moment, but we are going to change it soon. 

You may also notice why does that model is called `Conversation` instead of `Room`. `Room` should make more sense. 
You are right. Well, partially. We will get to that question soon as well.

## Working with a database

We have the rooms list in our application. The problem is they are static. Every time we want to update the list of the rooms we need to update the code. Also, there is no way to manage rooms by users of our application.

Nowadays almost every application has a persistent storage. Think of it as any kind of database. As you may know from [the first article](http://whatdidilearn.info/2018/01/14/phoenix-first-steps.html) of these series, We will use PostgreSQL for that application.

In the Phoenix framework, we are not working database directly. We are communicating with it through an additional layer called Ecto.

## Ecto

If you are familiar with other frameworks you probably know about [ORM (Object-relational Mapping](https://en.wikipedia.org/wiki/Object-relational_mapping) and might think: "Ah. The Ecto is exactly the same". Well, not completely. There are for sure some similarities.

If we take a look at ORM's description from [Wikipedia page](https://en.wikipedia.org/wiki/Object-relational_mapping):

> Object-relational mapping (ORM, O/RM, and O/R mapping tool) in computer science is a programming technique for converting data between incompatible type systems **using object-oriented programming languages**.

We know, Elixir is not an Object-Oriented Programming language. Probably what is why Ecto cannot be considered as ORM. Or does it?

Anyway, Ecto is a library and domain-specific language to interact with databases. It consists of 4 main components:

* `Repo` - the repository of data storage. Models hide implementation details from the rest of components. The Repo hides implementation details from models. It can provide access to different data storages such as databases or in-memory storages.
* `Schema` - Think of that as a mapping between a database table and its columns with the Elixir struct. It converts table's columns into Struct items.
* `Changeset` - Provides a way to cast attributes value into correct types, add validation rules and constraints.
* `Query` - Wrapper in Elixir around SQL queries.


Let's try to create a new table in our database to keep the information about rooms.

Ecto provides us migration generators, which we can use to describe the structure of the table and apply those changes in a database.

#### A side note about Phoenix generators

If you are using version 1.3 of the Phoenix framework then make sure you use `mix phx...` generators instead of `mix phoenix`. The last one will provide diffrent results in some cases, it is deprecated, and it would be removed in Phoenix 1.4. Don't train that muscle memory.

## Room model

Using Phoenix generators we can create our model as follows:

```
→ mix phx.gen.schema Conversation.Room rooms name description topic
```

From the `mix help phx.gen.schema`

> The resource fields are given using name:type syntax where type are the types supported by Ecto. Omitting the type makes it default to :string:

So generally it is the same as:

```
→ mix phx.gen.schema Conversation.Room rooms name:string description:string topic:string
```

Why do we use `Conversation.Room` instead of just `Room`? The `Conversation` module here would be a context.

### Contexts
  
Here we also need to stop a little bit and talk about contexts.

Phoenix 1.3 introduces the contexts functionality. Contexts help to group similar functionality under the same module. It can help to draw the borders between different modules. 

The main idea comes from the <a target="_blank" href="https://www.amazon.com/gp/product/0321125215/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321125215&linkCode=as2&tag=whatdidilearn-20&linkId=4e62f9505af13749863ed3c3f941efd8">Domain-Driven Design</a><img src="//ir-na.amazon-adsystem.com/e/ir?t=whatdidilearn-20&l=am2&o=1&a=0321125215" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" /> Book and also described as a [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html) by Martin Fowler.

You can also find more details what that decision has been made in the [ElixirConf 2017 Keynote](https://www.youtube.com/watch?v=tMO28ar0lW8) talk given by Chris McCord. The talk is quite interesting by the way. I would recommend to watch it.

So from now on, Phoenix requires us to group models into contexts.


Now since we are clear about contexts let's get back to our table. We have created it with the following columns.

* `name` - The name of the room
* `description` - The text which describes the intention of the room.
* `topic` - The current topic of the room can be changed from time to time, to reflect current state in the room.

Before we apply the migration itself we need to update it for our needs.

Let's update our migration to look as follows:

```elixir
defmodule Prater.Repo.Migrations.CreateRooms do
  use Ecto.Migration

  def change do
    create table(:rooms) do
      add :name, :string, null: false, size: 25
      add :description, :string
      add :topic, :string, size: 100

      timestamps()
    end
  end
end
```

We use `null: false` because a room should have a name. Also, the name should not be very long. So we limit it to 25 characters. That should be enough.
The topic of the room also should not be very long so we limit it to 100 characters.

Now we are ready to apply our migration by running:

```
→ mix ecto.migrate
```

It will create the `rooms` table in the database with the columns we have described. That is how or `Room` model looks now:

```elixir
defmodule Prater.Conversation.Room do
  use Ecto.Schema
  import Ecto.Changeset
  alias Prater.Conversation.Room


  schema "rooms" do
    field :description, :string
    field :name, :string
    field :topic, :string

    timestamps()
  end

  @doc false
  def changeset(%Room{} = room, attrs) do
    room
    |> cast(attrs, [:name, :description, :topic])
    |> validate_required([:name, :description, :topic])
  end
end
```

It has schema description and a changeset function already generated for us. Let's update the following line inf the `changeset` function:

```elixir
|> validate_required([:name, :description, :topic])
```

to 

```elixir
|> validate_required([:name])
```

Later we will understand the reason behind that change. Let's just proceed for now.


Now when we have (almost) everything set, let's run the Interactive Elixir and play a little bit with our `Room` model.

```elixir
iex> alias Prater.Conversation
iex> alias Prater.Conversation.Room
iex> alias Prater.Repo
```

Just a couple of aliases to minimize the amount of typing.

Now we can fetch the list of all rooms by calling:

```
iex> Repo.all(Room)
[debug] QUERY OK source="rooms" db=8.5ms queue=0.1ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 []
[]
```

We have used the `Repo` module by passing our model. We can also notice what SQL query does Echo produced for that.

There are no rooms in the database yet.

What if we call `changeset` function with empty attributes?

```elixir
iex> %Room{} |> Room.changeset(%{})
#Ecto.Changeset<
  action: nil,
  changes: %{},
  errors: [name: {"can't be blank", [validation: :required]}],
  data: #Prater.Conversation.Room<>,
  valid?: false
>
```

We can notice that our record is not valid because we are missing the name. We can also see the attributes set to `valid?: false` and `errors` contains the list of errors.

If we try to set the name, we can see that the record become valid.

```elixir
iex> %Room{} |> Room.changeset(%{name: "Lobby"})
#Ecto.Changeset<
  action: nil,
  changes: %{name: "Lobby"},
  errors: [],
  data: #Prater.Conversation.Room<>,
  valid?: true
>
```

When we have configured our schema we have specified the length limit for the `name` column, but our model does not know about that.

```elixir
iex> %Room{} |> Room.changeset(%{name: "That is a very very long name for the room"})
#Ecto.Changeset<
  action: nil,
  changes: %{name: "That is a very very long name for the room"},
  errors: [],
  data: #Prater.Conversation.Room<>,
  valid?: true
>
```

The name is definitely longer than 25 characters, but it is still considered as valid. Let's try to insert the record into database.


{% raw %}
```elixir
iex> %Room{} |> Room.changeset(%{name: "That is a very very long name for the room"}) |> Repo.insert()
[debug] QUERY ERROR db=47.9ms queue=0.2ms
INSERT INTO "rooms" ("name","inserted_at","updated_at") VALUES ($1,$2,$3) RETURNING "id" ["That is a very very long name for the room", {{2018, 1, 26}, {15, 38, 46, 991635}}, {{2018, 1, 26}, {15, 38, 46, 997210}}]
** (Postgrex.Error) ERROR 22001 (string_data_right_truncation): value too long for type character varying(25)
    (ecto) lib/ecto/adapters/sql.ex:554: Ecto.Adapters.SQL.struct/8
    (ecto) lib/ecto/repo/schema.ex:547: Ecto.Repo.Schema.apply/4
    (ecto) lib/ecto/repo/schema.ex:213: anonymous fn/14 in Ecto.Repo.Schema.do_insert/4
```
{% endraw %}

And now we can see it is failing.

To avoid that we can apply validation rules in the `changeset` function of our model.

### Apply validation rules

Remember we already had `validate_required([:name, :description, :topic])`? It was checking the presence of all these fields and we have shrunk it to only `:name`.

By using `validate_lenth(:name, min: 3, max: 25)` we are validating the length of the name field. We limit it to maximum of 25 characters, but also to minimum of 3. Rooms with the name shorter than 3 do not make much sense, they are not very descriptive.

Then we do the same with the `topic` field and limit the length of 5 and 100 characters.

Also, we are applying the unique constraint to the `name` fields by calling `unique_constraint(:name)`. We want to avoid duplicate rooms names. That can puzzle our users.

Here is how the whole function looks like:

```elixir
def changeset(%Room{} = room, attrs) do
  room
  |> cast(attrs, [:name, :description, :topic])
  |> validate_required([:name])
  |> unique_constraint(:name)
  |> validate_lenth(:name, min: 3, max: 25)
  |> validate_lenth(:topic, min: 5, max: 100)
end
```

Now let's recompile the code and play with our changes again.

{% raw %}
```
iex> recompile

iex> %Room{} |> Room.changeset(%{name: "Lobby", description: "The general chat room. Everybody welcome here."}) |> Repo.insert
[debug] QUERY OK db=48.3ms queue=0.1ms
INSERT INTO "rooms" ("description","name","inserted_at","updated_at") VALUES ($1,$2,$3,$4) RETURNING "id" ["The general chat room. Everybody welcome here.", "Lobby", {{2018, 1, 26}, {15, 46, 54, 682810}}, {{2018, 1, 26}, {15, 46, 54, 682891}}]
{:ok,
 %Prater.Conversation.Room{
   __meta__: #Ecto.Schema.Metadata<:loaded, "rooms">,
   description: "The general chat room. Everybody welcome here.",
   id: 1,
   inserted_at: ~N[2018-01-26 15:46:54.682810],
   name: "Lobby",
   topic: nil,
   updated_at: ~N[2018-01-26 15:46:54.682891]
 }}
```
{% endraw %}

We are creating the room using valid attributes and we see it was inserted into the database.

```
iex> Repo.all(Room)
[debug] QUERY OK source="rooms" db=16.4ms decode=0.2ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 []
[
  %Prater.Conversation.Room{id: 1, name: "Lobby", ...}
]
```

Let's try to add a room with the same name:

{% raw %}
```
iex> %Room{} |> Room.changeset(%{name: "Lobby"}) |> Repo.insert()
[debug] QUERY OK db=4.7ms
INSERT INTO "rooms" ("name","inserted_at","updated_at") VALUES ($1,$2,$3) RETURNING "id" ["Lobby", {{2018, 1, 26}, {15, 48, 21, 800726}}, {{2018, 1, 26}, {15, 48, 21, 800737}}]
{:ok, %Prater.Conversation.Room{id: 2, name: "Lobby" }}

iex> Repo.all(Room)
[debug] QUERY OK source="rooms" db=3.2ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 []
[
  %Prater.Conversation.Room{id: 1, name: "Lobby", ...},
  %Prater.Conversation.Room{id: 2, name: "Lobby", ...}
]
```
{% endraw %}

Oops. The insert was successful even though we had unique constraint. Why does that happen?

The thing is Ecto relays on unique indexes in a database. If the insert is failing because of violation of unique constraints, the changeset does not raise an error but returns the invalid record instead.

So let's try to fix that. We can either create a new migration to fix it or rollback our previous migration, update it and then migrate again.

As soon as we just create our rooms table and didn't release our feature yet, using rollback approach is totally fine. As long as we are still in the development phase of that feature. If you decide to add unique constraints to the table which already in use it should be done via new migration.

```
→ mix ecto.rollback
```

Now let's update the `change` function and add the following line after `create table ...` call.

```elixir
create unique_index(:rooms, [:name])
```

By doing that we are declaring unique index on the `rooms` table for the `name` column on the database side.

Now we need to apply our migration again.

```
→ mix ecto.migrate
```

Let's try to add duplicated rooms again.

{% raw %}
```
iex> Repo.all(Room)
[]
[debug] QUERY OK source="rooms" db=16.0ms queue=0.1ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 []

iex> %Room{} |> Room.changeset(%{name: "Lobby"}) |> Repo.insert()
[debug] QUERY OK db=8.7ms queue=0.1ms
INSERT INTO "rooms" ("name","inserted_at","updated_at") VALUES ($1,$2,$3) RETURNING "id" ["Lobby", {{2018, 1, 26}, {16, 0, 40, 216518}}, {{2018, 1, 26}, {16, 0, 40, 216618}}]
{:ok, %Prater.Conversation.Room{id: 1, name: "Lobby" ... }}

iex> %Room{} |> Room.changeset(%{name: "Lobby"}) |> Repo.insert()
[debug] QUERY ERROR db=7.6ms
INSERT INTO "rooms" ("name","inserted_at","updated_at") VALUES ($1,$2,$3) RETURNING "id" ["Lobby", {{2018, 1, 26}, {16, 0, 46, 349428}}, {{2018, 1, 26}, {16, 0, 46, 349450}}]
{:error,
 #Ecto.Changeset<
   action: :insert,
   changes: %{name: "Lobby"},
   errors: [name: {"has already been taken", []}],
   data: #Prater.Conversation.Room<>,
   valid?: false
 >}
```
{% endraw %}

The first time we have succeeded. Because we have no records in the database. But the second time we see the validation error `[name: {"has already been taken", []}]`. So now it works.

Let's query the database again to check if we still have a single record.

```
iex> Repo.all(Room)
[debug] QUERY OK source="rooms" db=2.7ms decode=0.1ms
SELECT r0."id", r0."description", r0."name", r0."topic", r0."inserted_at", r0."updated_at" FROM "rooms" AS r0 []
[
  %Prater.Conversation.Room{id: 1, name: "Lobby", ...}
]
```

And we have.

Let's create one more room. We will use it later.

```
iex> %Room{} |> Room.changeset(%{name: "Elixir"}) |> Repo.insert()
```

Now we have our model up and ready. It even has the records in the database. It is about time to display the data on the page.

We need to change the implementation of `Conversation.list_rooms` to use the `Room` model instead.

```elixir
defmodule Prater.Conversation do
  alias Prater.Repo
  alias Prater.Conversation.Room

  def list_rooms do
    Repo.all(Room)
  end
end
```

Just a couple of aliases to keep names shorter and now `list_rooms` function gets the data from the model.

<p align="center">
  <img src="{{ site.url }}/img/posts/ecto_and_models/two_rooms_list.png" />
</p>

As you can see on the page, that works perfectly fine.

## Wrapping up

We have covered a lot today. First, we have created the model with a static data and then we replaced it with another model which fetches the data from the database.
Then we change the usage of the static model to DB model without changing our template nor controller.

During these changes, we have learned what is Ecto and how to generate migrations.
We have talked about contexts and the reason behind them.

You can find all these changes on [the GitHub page](https://github.com/ck3g/prater/pull/2) of the app.

That are still basics. There is more stuff awaits.
