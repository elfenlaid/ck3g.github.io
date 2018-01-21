---
layout: post
title: "Controllers, Views and Templates"
desc: "The introduction to Controller, Views and Templates in Phoenix framework"
keywords: "Elixir, Phoenix, Controller, View, Template, Layout"
tags: [Elixir, Phoenix]
---

Last time we have covered the very basics of Phoenix framework.
We have managed to install a development environment and create the new project.
Today we will proceed and build the basic application layout.

As soon as we are building the chat application it would be nice to have rooms to separate conversations between users by topics or by groups of people.

We are going to build a layout similar to the following mockup.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/room_index_mockup.png" />
</p>

We have navigation bar there with the logo on the top of the page.
The list of the rooms on the left side and the working area in the middle.

To do that we need to become familiar with the views and controllers.
What exactly are views and controllers?

## MVC

The Phoenix framework implements the [MVC architectural pattern](https://en.wikipedia.org/wiki/Model–view–controller). Where MVC stands for Model-View-Controller.
MVC pattern splits the application into three components.
Those components have their own mission and mostly hide private details from others.

The **Model** provides the raw data for the application. It hides the details of a data source.

The **View** uses that data to nicely display it. The view does not care where the data comes from. It only cares about a rendering of that data. To present it as a table or as a list.

The **Controller** manipulates views and models. It figures out what does a user need, grabs the data from the model (if needed) and passes it to the view. The controller does not care where the data comes from and it does not care how the view will represent it. It simply says "Hey Model, please give me all available **rooms** you have". And then "Hey View, here are the rooms, please render those rooms on the page".

On the following picture (from [Wikipedia page about MVC](hhttps://en.wikipedia.org/wiki/Model–view–controller)) you can see the diagram of interactions within MVC pattern.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/MVC-Process.svg" />
</p>

Now as we understand the meaning of Views and Controllers we can proceed and implement the app interface.

## List of the rooms

We are going to display rooms list. So we need a corresponding controller for that. The controller in the Phoenix it is a module with additional functionality.
The controller, in turn, consists of actions. Actions are functions of that module, which represent, well, different actions.

By convention, Phoenix uses singular names for controllers followed by `Controller` word. In our case, it would be `RoomController`.

There is also convention regarding actions. We can name them whatever we want, but there are set of names it is recommended to pick from.

* `index` - Usually responsible for rendering a list of items
* `show` - Displays the particular item by its id
* `new` - Renders the form to create a new item
* `create` - Creates a new items (saves it in DataBase)
* `edit` - Renders the form to update existing item
* `update` - Updates an existing item
* `delete` - Deletes an existing item

Based on that, `index` is the best name to use for rendering the list of rooms.

To see a page, a user needs to visit particular URL. That URL should be somehow linked with the particular controller action.
Phoenix application has a `Router` module which describes URL routes. Which, in turn, matches the request with the controller action. We will talk about Router in one of the following articles. For now, let's just update our root URL to point to our new controller.

In the `lib/prater_web/router.ex` file we need to replace the following line

```elixir
get "/", PageController, :index
```

with 

```elixir
get "/", RoomController, :index
```

By doing that, we are linking the GET HTTP request of the root URL (in our case it's "http://localhost:4000") with the action `index` of the `RoomController`.

Now start the server using `mix phoenix.server` and open a root page. It will greet us with the following error:

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/controller_is_not_avaialble.png" />
</p>

That is correct. We don't have the controller yet.

Controllers located in `web/controllers` directory (at least for version 1.3 which I am using at the moment).

So let's go and create a `lib/prater_web/controllers/room_controller.ex` file with the following content:

```elixir
defmodule PraterWeb.RoomController do
end
```

We have defined the module for our controller. If we go back to the page, what would we see?

```
function PraterWeb.RoomController.init/1 is undefined or private
```

Phoenix tries to use that module as a controller, but some functionality is missing. That means we need to extend the module to add that functionality.

```elixir
defmodule PraterWeb.RoomController do
  use PraterWeb, :controller
end
```

We have extended the module so it can behave like a controller.

Go back to the page:

```
function PraterWeb.RoomController.index/2 is undefined or private
```

Now we need to define an action inside our controller.

```elixir
def index(conn, _params) do
end
```

The actions in Phoenix receive two arguments.
The first one is always `conn` which hold an information about the current request.
The second argument is `params` which is a map contains parameters passed with that HTTP request.

Now we see following error on the page:

```
expected action/2 to return a Plug.Conn, all plugs must receive a connection (conn) and return a connection
```

Our action does not do anything. That we need to do is to render the template.
We do that by calling `render/2` function providing connection and the template name:

```elixir
def index(conn, _params) do
  render conn, "index.html"
end
```

Following our errors.

```
function PraterWeb.RoomView.render/2 is undefined (module PraterWeb.RoomView is not available)
```

We find out we need to define a view. You can also see that Phoenix expects that view to be named as `RoomView` because we are using `RoomController`. That is also the convention.

Let's create the file `lib/prater_web/views/room_view.ex` and define a module inside:

```elixir
defmodule PraterWeb.RoomView do
end
```

The error changes to 

```
function PraterWeb.RoomView.render/2 is undefined or private
```

That is because our view module also missing some functionality

```elixir
defmodule PraterWeb.RoomView do
  use PraterWeb, :view
end
```

Now the error contains a lot of information

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/could_not_render_index.png" />
</p>

We see that Phoenix looking for `index.html` template in the `lib/prater_web/templates/room` directory.
Phoenix knows that there is a `RoomView` but there are no corresponding templates we want to render.

It also says

```
No templates were compiled for this module.
```

What does that exactly mean? Let's try to figure out.

When we compile our Phoenix application, Phoenix walks through defined view modules.
For every module, it cuts out the `View` suffix and gets the name.
That name would be `Room` for our `RoomView` module.
Then it looks for `room` (lower case name) directory inside `lib/prater_web/templates/`.
Inside `room` directory it gets every template file and compiles into a `render` function inside `RoomView` module.
Which we have called from our `index` action.

So we still missing the template. Let's create the `room` directory and the `lib/prater_web/templates/room/index.html.eex` file inside it. Leave it empty for now.

The `eex` extension of the file stands for [Embedded Elixir](https://hexdocs.pm/eex/EEx.html). It allows embedding the code inside a string. We will talk about that more in the future articles.

Now we see the error has gone and we see a blank page.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/blank_rooms_page.png" />
</p>

## Layouts

There is also one thing it is worth to talk about. The layouts.
Currently, we have a blank template file, but our web page is not completely blank.
We still have Phoenix logo on top of the page. So where does that comes from?

If we look into `lib/prater_web/templates` folder we would see there is another folder `layout` with `app.html.eex` file inside. If open that file we can see there is a complete HTML template in that file. It has HTML tags such as `<html>`, `<head>`, `<body>`.
So this file represents the general layout for the application. If we want to have some common elements (such as logo or navigation menu) from page to page, we don't need to duplicate them on every page. We can keep it in the layout file.

Deep inside `<body>` tag we can see following excerpt:

```eex
<main role="main">
  <%= render @view_module, @view_template, assigns %>
</main>
```

That is the place where templates are being injected. All the stuff before and after would be the same from page to page.

Let's focus on the header of our application.

We remove whole `<header></header>` section from the layout.
Then right after `<body>` tag we insert following markup:

```html
<div class="navbar navbar-default navbar-fixed-top">
  <div class="container">
    <div class="navbar-header">
      <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </button>
      <a class="navbar-brand" href="/">Prater</a>
    </div>
  </div>
</div>
```

And the following CSS into `assets/css/app.css`:

```css
body {
  min-height: 2000px;
  padding-top: 70px;
}
```

As soon as Phoenix uses [Bootstrap v3.3.5](http://bootstrapdocs.com/v3.3.5/docs/) (at the time I am writing that) that should do the trick.

We also need to remove the following lines from `assets/css/phoenix.css`:

```css
/* Customize container */
@media (min-width: 768px) {
  .container {
    max-width: 730px;
  }
}
```

You should see the new header now.

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/new_header.png" />
</p>

There are few steps left to finish our layout. We want to see the list of the rooms on the left and working area in the middle. Let's do that.

To achieve that we need to update our `index.html.eex` file with following content:

```html
<div class="row">
  <div class="col-md-3">
    <h3>Rooms</h3>
    <ul class="list-group">
      <li class="list-group-item">Lobby</li>
    </ul>
  </div>
  <div class="col-md-9">
    <div class="jumbotron">
      <div class="page-header">
        <h2>Welcome to Prater</h2>
      </div>
      <div class="page-header">
        <h2><small>Choose the room on the left to join</small></h2>
        <h2><small>or <a href="" class="btn btn-success">Create</a> a new one</small></h2>
      </div>
    </div>
  </div>
</div>
```

We are using hardcoded "Lobby" room here. But that is all we need for now.

That is how our page should look now:

<p align="center">
  <img src="{{ site.url }}/img/posts/phoenix_controllers_and_views/finished_layout.png" />
</p>

## Wrapping up

Today we have made a big step forward. We have got familiar with MVC pattern. We have walked the path from requesting a resource (by visiting certain URL) to the controller and its action, then to the rendered view and the HTML page displayed back to the user.

By learning that we have set the solid foundation for the next steps. It is important to understand how to work with controllers and views. By building web application you will use these pieces quite often.

See you in the next articles.

P.S. You can grab the code from [the GitHub page](https://github.com/ck3g/prater/pull/1).
