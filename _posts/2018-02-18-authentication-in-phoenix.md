---
layout: post
title: "Authentication in Phoenix"
desc: "Learn how to implement Authentication solution in Phoenix"
keywords: "Elixir, Phoenix, Auth, Authentication, Sign In, Sign Up, Sign out, login, register, logout"
tags: [Elixir, Phoenix]
excerpt_separator: <!--more-->
---

Authentication solutions are broad. They contain registration and login functionality, as well as Email confirmation, password recovery, update user's profile etc.

We are going to cover the most valuable part and implement registration and login functionality.

You can find some existing libraries for that. [Addict](https://github.com/trenpixster/addict) for example.
But we are learning, right? It would be better for us to understand how everything works under the hood.

So let's get started.

<!--more-->

## User model

We are going to work with users and allow them to use our application.
Then we need a `User` model for that. Let's start creating it by running the following generator.

```
→ mix phx.gen.schema Auth.User users email:unique username:unique encrypted_password
```

Before we migrate the DataBase let's prevent those fields to contain blank values.

```elixir
create table(:users) do
  add :email, :string, null: false
  add :username, :string, null: false
  add :encrypted_password, :string, null: false

  timestamps()
end
```

Now, we are ready to migrate our changes.

```
→ mix ecto.migrate
```

Our `User` model is in place, so we can proceed and implement the sign-in functionality.

## Sign In form

Let's start with the form. Our sign in form will be accessible under the following URL `http://localhost:4000/sessions/new`

If we try to open it now we will face the following error message.

```
no route found for GET /sessions/new (PraterWeb.Router)
```

We need a route, let's create it.

```elixir
# lib/prater_web/router.ex

scope "/", PraterWeb do
  # ... existing routes here
  resources "/sessions", SessionController, only: [:new, :create]
end
```

Next, we need a controller `lib/prater_web/controllers/session_controller.ex` with the following content:

```elixir
defmodule PraterWeb.SessionController do
  use PraterWeb, :controller

  def new(conn, _params) do
    render conn, "new.html"
  end
end
```

The view `lib/prater_web/views/session_view.ex`

```elixir
defmodule PraterWeb.SessionView do
  use PraterWeb, :view
end
```

and finally the template `lib/prater_web/templates/session/new.html.eex`

```html
<h1>Log In</h1>
```

If you are missing some parts you can review [the CRUD article](http://whatdidilearn.info/2018/02/04/implementing-crud-in-phoenix.html). There the importance of all these pieces is described there.

Now the page should be rendered without errors.

### Add SASS support

Before we move on to the actual implementation of the form. Let's add a new dependency to our project.
It would allow us to use SCSS extension for our styles. Although that is not mandatory.

Let's start from adding a Node.js dependency.

```
→ cd assets
→ npm install sass-brunch --save-dev
→ cd ..
```

Then we need to rename our `app.css` file to `app.scss`.

```
→ mv assets/css/app.css assets/css/app.scss
```

The next step is to update the `assets/brunch-config.js` file. We need to extend the `files.stylesheets` section with:

```
order: {
  after: ["priv/static/css/app.scss"]
}
```
That is pretty much it what we need to do to make it work.

We are ready to build the form

The Bootstrap 4 (which we are using by the way) comes with some examples.
We can take advantage of them to build our own sign in form from [the following example](https://getbootstrap.com/docs/4.0/examples/floating-labels/).

Let's update the `lib/prater_web/templates/session/new.html.eex` template with the following layout:

```erb
<div class="auth-form-wrapper">
  <%= form_for @conn, session_path(@conn, :create), [as: :session, class: "form-signin"], fn f -> %>
    <div class="text-center mb-4">
      <h1 class="h3 mb-3 font-weight-normal">Sign In</h1>
    </div>

    <div class="form-label-group">
      <%= text_input f, :email, class: "form-control", placeholder: "Email address", required: true, autofocus: true %>
      <%= label f, :email, "Email address", class: "control-label" %>
    </div>

    <div class="form-label-group">
      <%= password_input f, :password, class: "form-control", placeholder: "Password", required: true %>
      <%= label f, :password, class: "control-label" %>
    </div>

    <%= submit "Sign in", class: "btn btn-lg btn-primary btn-block" %>
  <% end %>
</div>
```

We have described the form layout here with two fields: Email and Password.

To see the form in our browser we need a `create` action in the controller.

```elixir
def create(conn, _params) do
end
```

The form is rendered now, but it does not look like the one from the example page.
To fix that we need to adjust some styles. Let's create a `assets/css/auth_form.scss` file with [the following content](https://github.com/ck3g/prater/blob/e70adbce0daf9b0fb7687a94e6d857d1279c06f9/assets/css/auth_form.scss).

Now the form looks nicer.

<p align="center">
  <img src="{{ site.url }}/img/posts/auth/sign_in_form.png" />
</p>


## Sign In functionality

We have a nice looking form, but we cannot use it yet. We are still missing the sign in functionality.

If we try to submit our form now, it would fail, because our `create` action is blank. Let's update it.

```elixir
def create(conn, %{"session" => %{"email" => email, "password" => password}}) do
  case Auth.sign_in(email, password) do
    {:ok, user} ->
      conn
      |> put_session(:current_user_id, user.id)
      |> put_flash(:info, "You have successfully signed in!")
      |> redirect(to: room_path(conn, :index))

    {:error, _reason} ->
      conn
      |> put_flash(:error, "Invalid Email or Password")
      |> render("new.html")
  end
end
```

Here we are pattern matching `email` and `password` fields submitted to the form.
Then we try to authenticate the user in the `Auth.sign_in/2`.
On success, we are storing the user's ID into session and redirect to the main page.
On error, we are rendering the form again and show an error message.

We still need to implement `Auth.sign_in/2` function.

Let's create the `lib/prater/auth/auth.ex` file with the following content:

```elixir
defmodule Prater.Auth do
  alias Prater.Repo
  alias Prater.Auth.User

  def sign_in(email, password) do
    user = Repo.get_by(User, email: email)

    cond do
      user && user.encrypted_password == password ->
        {:ok, user}
      true ->
        {:error, :unauthorized}
    end
  end
end
```

We have created our `Auth` context with the single `sign_in/2` function.
The function finds the user by a provided `email` and check if the user's password matches with the provided password.

Of course, keeping the password as a plain text is insecure and you should not do it any application. But are going to improve it soon.

To test our changes we need a user, otherwise how would be able so to sign in. Let's create one in the DataBase.

```
iex> %Prater.Auth.User{} |>
  Prater.Auth.User.changeset(%{email: "user@example.com", encrypted_password: "password", username: "user"}) |>
  Prater.Repo.insert()
```

Now we are ready, try to sign in with those credentials and you will be redirected to the room's page.

## Display username on the page

Now we have our user signed in, but there are no signs about that in the interface.

We can improve it and display the username on the page once he is signed in.


Update `lib/prater_web/templates/layout/app.html.eex` with the following layout.


```erb
<div class="d-flex flex-column flex-md-row align-items-center p-3 px-md-4 mb-3 bg-white border-bottom box-shadow">
  <h5 class="my-0 mr-md-auto font-weight-normal">
    <a href="/" class="navbar-brand text-dark"><strong>Prater</strong></a>
  </h5>
  <%= if Prater.Auth.user_signed_in?(@conn) do %>
    <nav class="my-2 my-md-0 mr-md-3">
      Signed in as: <strong><%= Prater.Auth.current_user(@conn).username %></strong>
    </nav>
  <% end %>
</div>
```

The only new code here is the `if` condition part with the content inside.

Now we need to implement following functions.

```elixir
def current_user(conn) do
  user_id = Plug.Conn.get_session(conn, :current_user_id)
  if user_id, do: Repo.get(User, user_id)
end

def user_signed_in?(conn) do
  !!current_user(conn)
end
```

After that, you should see a username in the top right corner of the page.

## Sign out functionality

As soon as our user can sign in, he should be able to sign out as well.

We need a new route for that.

```elixir
delete "/sign_out", SessionController, :delete
```

And the button. Let's add it next to the username.

```
<%= if Prater.Auth.user_signed_in?(@conn) do %>
  <nav class="my-2 my-md-0 mr-md-3">
    Signed in as: <strong><%= Prater.Auth.current_user(@conn).username %></strong>
  </nav>
  <%= link "Sign Out", to: session_path(@conn, :delete), method: :delete, class: "btn btn-outline-primary" %>
<% end %>
```

The button is ready but the `delete` action in `SessionController` still missing.

```elixir
def delete(conn, _params) do
  conn
  |> Auth.sign_out()
  |> redirect(to: room_path(conn, :index))
end
```

We are signing the user out and redirect back to the room's page.
The implementaion of the `Auth.sign_out/1` function looks like:

```elixir
def sign_out(conn) do
  Plug.Conn.configure_session(conn, drop: true)
end
```

We are dropping the whole session. That signs a user out.

Now, let's add small improvement and show "Sign In" button for signed out users

We need to add the following link to the `else` condition in the nav bar.

```
<%= link "Sign In", to: session_path(@conn, :new), class: "btn btn-outline-primary" %>
```

Now it is easier to navigate to sign in form.


## Make password encrypted

As I've already mentioned above, we have a plain password in the database which is unacceptable from the security point of view.
The secret credentials should be encrypted.

Let's fix that terrible mistake.

We are going to use [Comeonin](https://github.com/riverrun/comeonin) library to encrypt the password.

So at first, we need to add it to the project dependencies.

```elixir
defp deps do
  [
    # ...
    {:comeonin, "~> 4.0"},
    {:bcrypt_elixir, "~> 1.0"}
  ]
end
```

Starting from version 4 of Comeonin we need to choose the hashing algorithm.
If you have any preferences you can choose the one you like more. The Comeonin [wiki page](https://github.com/riverrun/comeonin/wiki/Choosing-the-password-hashing-algorithm) can help you decide.

I've chosen the `Bcrypt` for the sake of example.

Once you've added Comeonin to your dependencies run `→ mix deps.get`

We need a new user with hashed password. So let's drop our existing one and create it again. This time with the encrypted password:

```
iex> Prater.Repo.get_by(Prater.Auth.User, email: "user@example.com") |> Prater.Repo.delete()
iex> password = Comeonin.Bcrypt.hashpwsalt("password")
iex> %Prater.Auth.User{} |>
  Prater.Auth.User.changeset(%{email: "user@example.com", encrypted_password: password, username: "user"}) |>
  Prater.Repo.insert()
```

Now if we try to sign in we would get an error because we are not checking the password right.

Let's fix that. In the `Auth.sign_in/2` function we need to replace our current check:

```elixir
user.encrypted_password == password
```

with the

```elixir
Comeonin.Bcrypt.checkpw(password, user.encrypted_password)
```

That should make our sign in functionality work again.


Great. So now our user can sign in, sign out and has a secure password.
What we also want to have is the registration functionality.
So users will be able to register themselves and we don't need to create new records from the `iex` manually.


## Registration form

Similar to sign in functionality we need routes, controller, view, and templates.

First, come the routes:

```elixir
resources "/registrations", RegistrationController, only: [:new, :create]
```

Now we can add a link to registration page next to our "Sign in" link in the navigation bar.

```
<%= link "Sign Up", to: registration_path(@conn, :new), class: "btn btn-outline-primary ml-md-3" %>
```

Then we create a controller `lib/prater_web/controllers/registration_controller.ex` with the following content


```elixir
defmodule PraterWeb.RegistrationController do
  use PraterWeb, :controller
  alias Prater.Auth

  def new(conn, _params) do
    render conn, "new.html", changeset: conn
  end

  def create(conn, _params) do
  end
end
```


The view `lib/prater_web/views/registration_view.ex`

```elixir
defmodule PraterWeb.RegistrationView do
  use PraterWeb, :view
end
```

and the template itself `lib/prater_web/templates/registration/new.html.eex`

Fill it in with the complete layout of the form:

```erb
<div class="auth-form-wrapper">
  <%= form_for @changeset, registration_path(@conn, :create), [as: :registration, class: "form-signin"], fn f -> %>
    <div class="text-center mb-4">
      <h1 class="h3 mb-3 font-weight-normal">Sign Up</h1>
    </div>

    <div class="form-label-group">
      <%= text_input f, :email, class: "form-control", placeholder: "Email address", required: true, autofocus: true %>
      <%= label f, :email, "Email address", class: "control-label" %>
      <%= error_tag f, :email %>
    </div>

    <div class="form-label-group">
      <%= text_input f, :username, class: "form-control", placeholder: "User name", required: true %>
      <%= label f, :username, "User name", class: "control-label" %>
      <%= error_tag f, :username %>
    </div>

    <div class="form-label-group">
      <%= password_input f, :password, class: "form-control", placeholder: "Password", required: true %>
      <%= label f, :password, class: "control-label" %>
      <%= error_tag f, :password %>
    </div>

    <div class="form-label-group">
      <%= password_input f, :password_confirmation, class: "form-control", placeholder: "Password confirmation", required: true %>
      <%= label f, :password_confirmation, "Password confirmation", class: "control-label" %>
      <%= error_tag f, :password_confirmation %>
    </div>

    <%= submit "Sign Up", class: "btn btn-lg btn-primary btn-block" %>
  <% end %>
</div>
```

Comparing to Sign In form, that form has two more fields such as "Username" and "Password confirmation".

If you navigate to "/registrations/new" or click the "Sign Up" button then you should see the form.

<p align="center">
  <img src="{{ site.url }}/img/posts/auth/sign_up_form.png" />
</p>


## Register functionality

Our `create` action is now blank. Let's update it with the following code:

```elixir
def create(conn, %{"registration" => registration_params}) do
  case Auth.register(registration_params) do
    {:ok, user} ->
      conn
      |> put_session(:current_user_id, user.id)
      |> put_flash(:info, "You have successfully signed up!")
      |> redirect(to: room_path(conn, :index))

    {:error, changeset} ->
      render(conn, "new.html", changeset: changeset)
  end
end
```

Here we grab the params submitted through the form. We are trying to register a user.
In a successful case, we are signing him in and redirect to the room's page.
We are showing registration form back in case of errors.

The next step would be the implementation of 'Auth.register/1'.

```elixir
def register(params) do
  User.registration_changeset(%User{}, params) |> Repo.insert()
end
```

We use submitted params and pass it through `registration_changeset` and try to create a new record in the DataBase.

Now, let's go to the `User` model and implement all required changes there.

First, we need to slightly update our `User.changeset` to look like:

```elixir
@doc false
def changeset(%User{} = user, attrs) do
  user
  |> cast(attrs, [:email, :username])
  |> validate_required([:email, :username])
  |> validate_length(:username, min: 3, max: 30)
  |> unique_constraint(:email)
  |> unique_constraint(:username)
end
```

We require only `email` and `username` here and also adding length validation for the `username`.
We have skipped the `password` param because we will work with that separately.

Let's move on to `User.registration_changeset`:

```elixir
@doc false
def registration_changeset(%User{} = user, attrs) do
  user
  |> changeset(attrs)
  |> validate_confirmation(:password)
  |> cast(attrs, [:password], [])
  |> validate_length(:password, min: 6, max: 128)
  |> encrypt_password()
end
```

We are adding some validation rules on top of `User.changeset`.
Then we are validating the password confirmation, we want a user to confirm a password from the registration form.
Then we clear up the password because we are not going to save it in the DataBase.
But we also validate the length of it to be within 6 and 128 characters.
As the last step, we are encrypting the password.

```elixir
defp encrypt_password(changeset) do
  case changeset do
    %Ecto.Changeset{valid?: true, changes: %{password: password}} ->
      put_change(changeset, :encrypted_password, Comeonin.Bcrypt.hashpwsalt(password))
    _ ->
      changeset
  end
end
```

If the changes are valid we are encrypting and updating the `encrypted_password` field.

We need the last piece to glue all together. We need to update the `schema` in the `User` model.

```elixir
schema "users" do
  # ...
  field :password, :string, virtual: true
  field :password_confirmation, :string, virtual: true
end
```

We are defining virtual fields to be able to use them in our changesets.

Try to check the registration form now, it should work.

That concludes our basic implementation.

## Wrapping up

Today we have made an additional step in our learning path. We have implemented most valuable pieces for any authentication solution.
We can register and new user, allow him to sign in and sign out.
Of course that there is a big room for improvements.
The improved implementation should contain additional components, such as Email confirmation, password recovery, ability to change the password or even sign in by username. It should also restrict users from the access of certain pages if they are not signed in.

But you can get the general idea from that post and proceed with those improvements.

The complete implementation we have made today, you can find on [the GitHub page of the project](https://github.com/ck3g/prater/pull/6).
