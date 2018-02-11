---
layout: post
title: "How to use Bootstrap 4 with Phoenix"
desc: "Learn how to integrate Bootstrap 4 into Phoenix project using CDN and npm.."
keywords: "Elixir, Phoenix, Bootstrap 4, CDN, npm, front-end"
tags: [Elixir, Phoenix]
---

[Bootstrap 4](http://getbootstrap.com/) has been out recently.
Let's use the advantage of that and learn how to integrate front-end libraries into Phoenix projects.


There are several ways to achieve that. Let's take a look at some of them.

## Use CDN

The first approach we are going to cover is to use Content Delivery Network (CDN).

Many libraries nowadays provide a compiled version of itself hosted on CDN.
The advantage of that approach that it allows easily add to your project and it would be faster for users to get that file.
CDNs are distributed across the planet and work in the way that user who opens your site will hit closest CDN to him, even if your host server located on a different continent. It also caches the file, so if a user has been used the same library on the other site he would get that file even faster.

Check out the ["7 Reasons to use a CDN"](https://www.sitepoint.com/7-reasons-to-use-a-cdn/) article for more stuff about the subject.

Every approach has its disadvantages. So does and CDN.

It requires an internet connection to work. So it depends how much do you work without internet on your computer.

You are depending on external service. Although the chance of that service becomes unavailable is quite rare.

It is also a matter of trust to the resource which serves files to you. Will it be up? Will it play by rules and won't inject harmful code via those files?
Those are the questions you need to ask before deciding to use it.

Anyway, let's move to implementation.

To starting using Bootstrap from CDN we need to update our layout file `lib/prater_web/templates/layout/app.html.eex` by adding CSS to the `<head>` right before we are using our `css/app.css` file.

```html
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">
<!-- Before that line -->
<link rel="stylesheet" href="<%= static_path(@conn, "/css/app.css") %>">
```

Here we start using the Bootstrap CND link. You can also see there is `integrity` option. But what does it mean?

I will quote the description from [MDN web docs](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) but I would also recommend reading the document as well.

> Subresource Integrity (SRI) is a security feature that enables browsers to verify that files they fetch (for example, from a CDN) are delivered without unexpected manipulation. It works by allowing you to provide a cryptographic hash that a fetched file must match.

That can reduce one of your fears and project yourself from getting a harmful version of the file.

Next, let's add JavaScript files. Bootstrap 4 depends on jQuery and Popper.js. So we need to add those dependencies in front of Bootstrap JS file.
We also put all these lines before the usage of our `js/app.js`.

```html
<script src="https://code.jquery.com/jquery-3.2.1.slim.min.js" integrity="sha384-KJ3o2DKtIkvYIK3UENzmM7KCkRr/rE9/Qpg6aAZGJwFDMVNA/GpGFF93hXpG5KkN" crossorigin="anonymous"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.12.9/umd/popper.min.js" integrity="sha384-ApNbgh9B+Y1QKtv3Rn7W3mgPxhU9K/ScQsAP7hUibX39j7fakFPskvXusvfa0b4Q" crossorigin="anonymous"></script>
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js" integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRRSQQxSfFWpi1MquVdAyjUar5+76PVCmYl" crossorigin="anonymous"></script>
<!-- Before that line -->
<script src="<%= static_path(@conn, "/js/app.js") %>"></script>
```

That is pretty much it. We have Bootstrap 4 in our projects. Although the layout differs from Bootstrap 3 we had before. So we need to update our layout to work with Bootstrap 4. Let's dot hat.


## Make layout work

First, we can remove following files which come with Phoenix. We don't need them anymore.

```
→ rm assets/css/phoenix.css assets/static/images/phoenix.png
```

Remove following lines from `assets/css/app.css`

```
body {
  min-height: 2000px;
  padding-top: 70px;
}
```

Next, in the `lib/prater_web/templates/layout/app.html.eex` file we need to replace the "navbar" section with the following markup.

```html
<div class="d-flex flex-column flex-md-row align-items-center p-3 px-md-4 mb-3 bg-white border-bottom box-shadow">
  <h5 class="my-0 mr-md-auto font-weight-normal">
    <a href="/" class="navbar-brand text-dark"><strong>Prater</strong></a>
  </h5>
</div>
```

We also need to wrap our flash messages into conditions to render them only when we have some of them.

```html
<%= unless is_nil(get_flash(@conn, :info)) do %>
  <p class="alert alert-success" role="alert"><%= get_flash(@conn, :info) %></p>
<% end %>
<%= unless is_nil(get_flash(@conn, :error)) do %>
  <p class="alert alert-danger" role="alert"><%= get_flash(@conn, :error) %></p>
<% end %>
```

as the last piece we need to update our jumbotron section in the `lib/prater_web/templates/room/index.html.eex` file with:

```html
<div class="col">
  <section class="jumbotron text-center">
    <h1 class="jumbotron-heading mb-5">Welcome to Prater</h1>
    <p class="lead text-muted">
      Choose the room on the left to join
    </p>
    <p class="lead text-muted">
      or <a href="" class="btn btn-success">Create</a> a new one
    </p>
  </section>
</div>
```

The complete changes of the layout update you can find in [the following commit](https://github.com/ck3g/prater/commit/3702e8698218f378831367591d1dc16eb31096a5).

That should be enough for our layout to support Bootstrap 4.

Now let's check if our JS works. We are going to add a small tooltip message to our rooms' list.

In the `lib/prater_web/templates/room/index.html.eex` file, let's replace the room's link

```html
<%= link room.name, to: room_path(@conn, :show, room.id) %>
```

with the following code

```html
<%= link room.name, to: room_path(@conn, :show, room.id), title: room.description, data: [toggle: "tooltip", placement: "right", html: "true"] %>
```

Here we are describing that the link will have a popover tooltip. To enable tooltips in the project we also need to add following lines into `assets/js/app.js`:

```js
$(function () {
  $('[data-toggle="tooltip"]').tooltip()
})
```
And it should work.

<p align="center">
  <img src="{{ site.url }}/img/posts/bootstrap4/popover.png" />
</p>

In the previous versions Bootstrap used to ship with icons, now it does not include them anymore and we need to use external libraries. Bootstrap also provides [a list of libraries](https://getbootstrap.com/docs/4.0/extend/icons/) we can use.

Back in the day, I was using a collection of the icons called [Font Awesome](https://fontawesome.com/). Let's use them to add small improvements to our interface.

As [it stated](https://fontawesome.com/get-started) on Font Awesome page, the easiest and recommended way to add that library into our project is to use CDN. We already familiar with that one. Let's add a CDN link right after our Bootstrap CDN.

```html
<script defer src="https://use.fontawesome.com/releases/v5.0.6/js/all.js"></script>
```

Now we are ready to use icons.

The general usage format is:

```HTML
<i class="fas fa-icon-name"></i>
```

Where the `icon-name` is the name of the icon you can find on the Font Awesome site.

Let's improve our tooltip by displaying a small "info" icon next to a room name. To do that we need to update the `<li>` tag in the `lib/prater_web/templates/room/index.html.eex` file.

```html
<li class="list-group-item">
  <%= link room.name, to: room_path(@conn, :show, room.id) %>
  <%= if !is_nil(room.description) && room.description != "" do %>
    <div class="float-right text-info" data-toggle="tooltip" data-placement="right" data-html="true" title="<%= room.description %>">
      <i class="fas fa-info-circle"></i>
    </div>
  <% end %>
</li>
```

We are drawing the icon here on the right side of our list item. By hovering the cursor over that icon we are showing a description of the room. We are also drawing the icon only for rooms which have a description.

<p align="center">
  <img src="{{ site.url }}/img/posts/bootstrap4/info_icon_tooltip.png" />
</p>

Ok, I think we did enough to check that the libraries are working.

Let's try a different approach to adding libraries to the project.



## Use npm

npm is the package manager for JavaScript. It can be used to install different dependencies such as Bootstrap.

First, we need to remove CDN links to jQuery, Popper.js, and Bootstrap we were using in the beginning.

Next, in the `assets/package.json` file we need to add those libraries into "dependencies" section.

```js
"bootstrap": "4.0.0",
"jquery": "3.2.1",
"popper.js": "^1.12.9",
```

We also would need to add a "sass-brunch" as a "devDependencies" in the same file.

```js
"sass-brunch": "^2.10.4",
```

Because we are going to use SCSS to serve Bootstrap files.

Now, we are ready to install those dependencies.

```
→ cd assets
→ npm install
→ cd -
```

The next step is to update the `assets/brunch-config.js` file. We need to extend the `files.stylesheets` section with:

```js
order: {
  after: ["priv/static/css/app.scss"]
}
```

add following lines into `plugins` section

```js
sass: {
  options: {
    includePaths: ["node_modules/bootstrap/scss"],
    precision: 8
  }
}
```

and extend `npm` section with the following configuration:

```js
globals: {
  $: 'jquery',
  jQuery: 'jquery',
  bootstrap: 'bootstrap'
}
```

We almost finished. The last thing we need to rename `app.css` to `app.scss`

```
→ mv assets/css/app.css assets/css/app.scss
```

and import Bootstrap in that file

```scss
@import "bootstrap";
```

That is pretty much it. Now we can restart the Phoenix server and check if everything still works. It does for me.


## CDN vs npm

I believe there are two camps of people who like either approach. You can probably find a lot of links where people arguing against each of these approaches.

Regarding to our case, I would say npm approach is trickier to install than use CDN links.

To be honest that took me a while to google and figure out how to setup it properly.

But it allows reusing Bootstrap's SCSS variables to have a consistent style.

For example:

```css
.jumbotron-heading {
  color: $orange;
}
```

would turn your Jumbotron header to Orange. Of course only if `$orange` still orange and not some kind of green.

In the end, it is up to you which approach to use. Now you know them both.

## One more way

There is one more way to add Bootstrap into your project. Well, technically two ways, but they are very similar by their nature.

You can also copy either already [compiled JavaScript and CSS](http://getbootstrap.com/docs/4.0/getting-started/download/#compiled-css-and-js) files or [source files](http://getbootstrap.com/docs/4.0/getting-started/download/#source-files) into the `assets/` directory. Then you can use them as the rest of your own assets.


## Wrapping up

Today we have figured out how to update the front-end libraries for the Phoenix project. We have migrated to newest (as for today) Bootstrap 4. We have covered two different approaches to integrate Bootstrap into our project by using CDN and npm. Using that knowledge now we can add additional libraries into the project by applying the same techniques.

You can find the complete code examples on the GitHub. [The CDN + Layout changes](https://github.com/ck3g/prater/pull/4) and [NPM approach](https://github.com/ck3g/prater/commit/1a1d3568656946b05135af022776cd03607422a5).

See you next time.
