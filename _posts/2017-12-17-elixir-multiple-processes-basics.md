---
layout: post
title: "Elixir. Multiple processes. Basics."
desc: "How to work with multiple processes in Elixir"
keywords: "Elixir, processes, concurrency, Spawning processes"
tags: [Elixir]
---

This time we will go deeper and get familiar with the concurrent programming in Elixir.
We will meet processes. Will learn what are they, why do we need them and how to work with them.

## What is a concurrency

Concurrency is the ability to execute several parts or instances of the same program at the same time.
In the same way, as you can have several applications running at the same time on your computer,
you can develop your application which can run some tasks in a concurrent way.
By the way, an instance of a running application is also a process.
That being said an application consists of 1+ processes.

Concurrency considered a difficult topic in other programming languages, but it is much simpler in Elixir.

## What is a process

As Wikipedia page says:

> In computing, a process is an instance of a computer program that is being executed. It contains the program code and its current activity. Depending on the operating system (OS), a process may be made up of multiple threads of execution that execute instructions concurrently.

Threads and processes have differences though.
Long story short, processes do not share the same resources with each other but threads do.
Threads are also subsets of a process.

Elixir concurrency is based on the actor model. That means processes do not share any data with each other.
They can send and receive messages, but that pretty much it.

Unlike other programming languages, processes in Elixir are managed by Erlang's VM.
That makes them lightweight and easy to run thousands of them at the same time.


## When to use processes

Elixir, like many other functional languages, are stateless by nature and variables are immutable.
In order to keep the state somehow, it requires passing values through the functions and chain functions together.

If we take web request as an example we can see roughly describe it as following.

It is stateless by nature as well. When a user visits a page, a browser sends a request to a server.
A server does all required stuff, like querying a DataBase, preparing views, etc.
Then renders page back to a user.
That is it.
Once a user opens a new page, all that process is starting all over again.
In that case, the state is kept in the DataBase or Cookies etc.

In the different type of applications, such as multiplayer games or chat apps, it can be more complicated to keep the state in the DataBase or some other places alike.

The processes can become handy for these cases.

Thus we can use processes when we need to split the work and do it in parallel or we want to keep the state.
The processes can also communicate with each other by sending and receiving messages.

Now let's take a look how to work with processes in Elixir.

## Basic functionality

Let's get the simple "Hello, world" application which we can generate by running `mix new hello_world`.
After tiny changes, this application contains the following function.

```elixir
defmodule HelloWorld do
  def hello do
    IO.puts "Hello, world!"
  end
end
```

The function prints "Hello, world!" on the screen:

```elixir
> HelloWorld.hello
Hello, world!
:ok
```

Now let's try to run it in a separate process.

## Spawn a process

To spawn a new process there are `Kernel.spawn/1` and `Kernel.spawn/3` functions.
The difference is that first function receives an anonymous function as an argument.
The second function receives module name, function name as an atom, and the list of function arguments.

```elixir
iex> spawn(HelloWorld, :hello, [])
Hello, world!
#PID<0.143.0>

iex> spawn(fn -> HelloWorld.hello end)
Hello, world!
#PID<0.145.0>
```

We can see, our function has been called and we got the **P**rocess **ID**entifier back.

As I've mentioned above the processes can communicate using messages between each other.

## Sending and receiving messages

Let's try to send a message to one process, that process will receive the message and send a new message back.

First, we need to update our `hello/1` function to start listening to new messages.

```elixir
defmodule HelloWorld do
  def hello do
    receive do
      {pid, name} -> send(pid, {:ok, "Hello, #{name}!"})
    end
  end
end
```

Here we use `receive` to listening for a message. Then we use `Kernel.send/2` to send a message back.
`Kernel.send/2` receives a destination as a first argument, which is PID in our case, and a message content as a second argument. The message can be any type, in this case, I am using a tuple to easily match it later.
I guess it is also considered as a best practice in Elixir.

Now, let's fire up our Interactive Elixir and check it out.

```elixir
iex> pid = spawn(HelloWorld, :hello, [])
#PID<0.125.0>

iex> send pid, {self(), "message"}
{#PID<0.110.0>, "message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
"Hello, message!"
```

As a first step, we are spawning a new process and capture its PID.

Then we send a message to that process passing PID of a calling process and a "name". 
In our case calling process is `iex`, so we pass PID of `iex` session.

Now as soon as `HelloWorld.hello/0` sent us the message back, all we need to do is to receive that message.
And we did.

Great. Now, without leaving an `iex` let's try to send the second message.

```elixir
iex> send pid, {self(), "second message"}
{#PID<0.110.0>, "second message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
```

What we've got here is handing `iex` session.
When we send a message for a first time, our `HelloWorld.hello/0` receives it and then exits.
What is why there is no one to respond to a second message.

We can fix that behavior by setting a timeout for `receive`. We can specify the timeout in the `after` section.

```elixir
iex> pid = spawn(HelloWorld, :hello, [])
#PID<0.123.0>

iex> send pid, {self(), "first message"}
{#PID<0.121.0>, "first message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
"Hello, Process!"

iex> send pid, {self(), "second message"}
{#PID<0.121.0>, "second message"}

iex> receive do
...>   {:ok, msg} -> msg
...> after 1_000 -> "Time is up!"
...> end
"Time is up!"
```

The first message we have received as we did before.
For the second message, we specify the timeout for one second.
Because of that, we have got "Time is up!" message back after one second.

That is cool. At least we are not hanging anymore. But we still want our `hello` function to handle multiple messages.
How would we do that? We probably should listen to a messages over and over again.

```elixir
defmodule HelloWorld do
  def hello do
    receive do
      {pid, name} ->
        send(pid, {:ok, "Hello, #{name}!"})
        hello()
    end
  end
end
```

Recursion is our friend here. Once we have received a message, we respond back and call the `hello` function again.

```elixir
iex> pid = spawn(HelloWorld, :hello, [])
#PID<0.147.0>

iex> send pid, {self(), "first message"}
{#PID<0.121.0>, "first message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
"Hello, first message!"

iex> send pid, {self(), "second message"}
{#PID<0.121.0>, "second message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
"Hello, second message!"

iex> send pid, {self(), "third message"}
{#PID<0.121.0>, "third message"}

iex> receive do
...>   {:ok, msg} -> msg
...> end
"Hello, third message!"
```

## Linking processes

Let's try to break our process by sending an unexpected result to `hello` function.
The function does not expect to receive anonymous function.

```elixir
iex(1)> self
#PID<0.110.0>

iex(2)> pid = spawn(HelloWorld, :hello, [])
#PID<0.113.0>

iex(3)> send pid, {self, fn -> "Error" end}
{#PID<0.110.0>, #Function<20.99386804/0 in :erl_eval.expr/5>}

18:31:40.163 [error] Process #PID<0.113.0> raised an exception
...


iex(4)> send pid, {self, "Not an error"}
{#PID<0.110.0>, "Not an error"}

iex(5)> receive do
...(5)>   {:ok, msg} -> msg
...(5)> after 1_000 -> "No response"
...(5)> end
"No response"

iex(6)> self
#PID<0.110.0>
```

At first, we call `self` just to capture the PID of the parent process (`iex` in our case).
Then we spawn a process as usual but pass an anonymous function as a name.
As the result, the spawned process dies.

Then we try to send a message to a process, but it didn't respond back.

Then I call the `self` one more time just to show that we are in the same parent process.

If we want processes to know about "problems" of each other we can link them using `spawn_link` functions.
Let's take a look at the following example.

```elixir
iex(1)> self
#PID<0.110.0>

iex(2)> pid = spawn_link(HelloWorld, :hello, [])
#PID<0.125.0>

iex(3)> send pid, {self, fn -> "Error" end}

18:34:21.774 [error] Process #PID<0.125.0> raised an exception
...

Interactive Elixir (1.5.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> self
#PID<0.127.0>

iex(2)> pid
** (CompileError) iex:2: undefined function pid/0
```

This time once we make our process crash we can see that the linked parent process was restarted (you can see it by the different PID) and all the process has been started over.

## Keeping a state

In the beginning, I've also mentioned that we can use processes to keep some sort of state.
Let's get our Toy Robot from [the previous articles](http://whatdidilearn.info/tags#Toy-Robot) and apply the knowledge we just get to make it a stateful robot.

```elixir
defmodule ToyRobot.StatefulRobot do
  alias ToyRobot.StatefulRobot

  def place do
    {:ok, state} = ToyRobot.place
    pid = spawn_link(fn -> listen(state) end)
    Process.register(pid, StatefulRobot)
    pid
  end

  def listen(state) do
    receive do
      {:report, pid} ->
        send(pid, state)
        listen(state)
      {:move} -> ToyRobot.move(state) |> listen
      {:left} -> ToyRobot.left(state) |> listen
      {:right} -> ToyRobot.right(state) |> listen
    end
  end

  def move, do: send(StatefulRobot, {:move})

  def left, do: send(StatefulRobot, {:left})

  def right, do: send(StatefulRobot, {:right})

  def report do
    send(StatefulRobot, {:report, self()})

    receive do
      state -> ToyRobot.report(state)
    end
  end
end
```

Once we place our robot, we capture its position and spawn a new process using `listen` function.
The function "listens" to new messages, gives orders to a robot based on those messages and call itself again passing the updated position.

Movement functions only send a new message all the time. The `report` function sends a message to a process and receives back an answer. The answer contains the current position of the robot.

```elixir
iex> alias ToyRobot.StatefulRobot, as: Robot
iex> Robot.place
iex> Robot.move
iex> Robot.right
iex> Robot.move
iex> Robot.move
iex> Robot.move
iex> Robot.report
{3, 1, :east}
```

The complete changes you can find in [this pull request](https://github.com/ck3g/toy_robot/pull/2) on a GitHub page.

## Wrapping up

I don't know about you, but for me, concurrency subject was a scary one until now. I was always trying to avoid it.

Now with Elixir, it does not look like that anymore. We can see how simple it can be.
Of course, it is only a beginning of a "concurrency path". We will see more interesting stuff in subsequent articles.
