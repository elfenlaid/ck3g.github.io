---
layout: post
title: "Elixir - first steps"
desc: "I am making first steps into Elixir. Learn how to install and write my first Hello World program"
keywords: "First steps, Elixir, Phoenix, learning Elixir, install, mix, hello world"
tags: [Elixir]
---

What should people, who starting to learn a new language or technology, start from? Of course from implementing a classic "Hello World" project and try it out. Let's try to do this as well.

<br />

First of all, we need to figure out how to install Elixir. In most of the cases that process is extremely easy. For macOS using Homebrew, it's literally one line in the terminal.

```
brew install elixir
```

<br />

Yes. That is pretty much it.

For the other operating systems that process does not differ a lot. You can find more examples on the [official page](https://elixir-lang.org/install.html).

<br />

Having Elixir installed on the machine we can it in action right away.

By running `iex` (stands for Interactive Elixir) from the terminal window we can start interactive mode. In that mode, we can type any Elixir expression and see a result of it. That is the same as `irb` for those who are familiar with Ruby programming language.

```elixir
iex(1)> 1 + 2
3
iex(2)> name = "John"
"John"
iex(3)> "Hello, " <> name
"Hello, John"
```

To leave the Interactive Elixir, you need to hit Ctrl+C twice.

<br />

Now let's try to write our simplest version of Hello World program.
To do that we need to create a `*.exs` file with the following content:

```elixir
IO.puts "Hello, World!"
```

This program prints out "Hello, World!" text into your terminal window. To check it out we need to use `elixir` executable:

```
→ elixir hello_world.exs
Hello, World!
```

We can also compile our file from the Interactive Elixir:

```
→ iex
Erlang/OTP 20 [erts-9.0.4] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.5.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> c "hello.exs"
Hello, World!
[]
```

<br />

Now it would be nice to go a little bit deeper and try to build an improved version of Hello World program. We can use `mix` tool to help us achieve our goal.

<br />

#### What is `mix`?

As [official Elixir site](https://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html) says:

> **Mix** is a build tool that ships with Elixir that provides tasks for creating, compiling, testing your application, managing its dependencies and much more

For those who are familiar with Ruby, `mix` is some kind of mixture of `Bundler`, `rake` and a little bit of Ruby on Rails generators.

It can be represented as the following diagram:

```
Elixir
 └── mix
     ├── Create applications
     ├── Compile applications
     ├── Run tasks (such as tests)
     └── Manage dependencies
```

<br />

Let's try to generate new application by typing `mix new hello_world` in the terminal window:

```
→ mix new hello_world
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/hello_world.ex
* creating test
* creating test/test_helper.exs
* creating test/hello_world_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd hello_world
    mix test

Run "mix help" for more commands.
```

<br />

Mix generates a bunch of files for us, but more interesting are:

`mix.exs` - contains application version and dependencies<br />
`lib/hello_world.ex` - our actual implementation of the project<br />
`text/hello_world_test.exs` - contains tests for the implementation file<br />

<br />

By skipping documentation comments (I just remove it for now), our `hello_world.ex` file has following content:

```elixir
defmodule HelloWorld do
  def hello do
    :world
  end
end
```

and test file for that is:

```elixir
defmodule HelloWorldTest do
  use ExUnit.Case
  doctest HelloWorld

  test "greets the world" do
    assert HelloWorld.hello() == :world
  end
end
```

<br />

We can test existing function of our application from the Interactive Elixir. To do that we need to run `iex` with `-S mix` param. That will tell `mix` to compile our project and start `iex`.


```
→ iex -S mix
Erlang/OTP 20 [erts-9.0.4] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiling 1 file (.ex)
Generated hello_world app
Interactive Elixir (1.5.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

<br />

Now we can call or function using `ModuleName.function_name` format:

```
iex(1)> HelloWorld.hello
:world
```

<br />

Let's try to run tests from the separate terminal window:

```
→ mix test
Compiling 1 file (.ex)
.

Finished in 0.03 seconds
1 test, 0 failures

Randomized with seed 248841
```

We even have a green test for our application.
Pretty cool, huh?

Now let's go deeper and try to update our code.
I want the `HelloWorld.hello` function to return "Hello, World!" string instead. We can achieve it by changing this function to be like:

```elixir
def hello do
  "Hello, World!"
end
```

Do you still have an `iex` running? Let's try to call this function again:

```elixir
iex(2)> HelloWorld.hello
:world
```

<br />

Exactly. We still have an old result. That is because `iex` does not recompile our code on the fly. But can ask him to do that by using `recompile` function:

```
iex(3)> recompile
Compiling 1 file (.ex)
:ok
iex(4)> HelloWorld.hello
"Hello, World!"
```

<br />

Now we have our changes up to date.
Let's try to run our tests again:

```
→ mix test
Compiling 1 file (.ex)


  1) test greets the world (HelloWorldTest)
     test/hello_world_test.exs:5
     Assertion with == failed
     code:  assert HelloWorld.hello() == :world
     left:  "Hello, World!"
     right: :world
     stacktrace:
       test/hello_world_test.exs:6: (test)



Finished in 0.04 seconds
1 test, 1 failure

Randomized with seed 954401
```

<br />

Now we can see our test is failing, and that does a perfect sense, because we have updated our production code, but did not change the test. Let's update our test assertion to check it against "Hello, World!" string instead of `:world`.

```elixir
test "greets the world" do
  assert HelloWorld.hello() == "Hello, World!"
end
```

and run the test:

```
→ mix test
.

Finished in 0.04 seconds
1 test, 0 failures

Randomized with seed 370635
```

And we are cool again.

That actually concludes our examples of the first steps in the world of Elixir.

<br />

What did we learn here? Now we know how to install Elixir on our computers, how to implement Hello World application in several ways, how to run that application (again) in several ways. How to use `mix` to help us achieve those goals.

The journey only begins. I believe there is more interesting stuff in the Elixir and I'm looking forward to getting to know them. 
I like that journey so far.

