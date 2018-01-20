---
layout: post
title: "Introduction to OTP, GenServers and Supervisors"
desc: "This time we are going to learn about OTP (Open Telecom Platform). We will get familiar with the GenServers and how to supervise processes using Supervisor module"
keywords: "Elixir, processes, OTP, Open Telecom Platform, GenServer, Supervisor"
tags: [Elixir]
---

In the previous articles, we have covered how to work with the processes in Elixir on a low level.
We have also covered what kind of wrappers exist around processes. That can help us to avoid writing that low-level code. Yes, I am mentioning [Agents and Tasks](http://whatdidilearn.info/2017/12/24/agents-and-tasks-in-elixir.html) now.
Now it's time to get familiar with OTP and its features called GenServer and Supervisors.
Let's dive in.


## What is OTP

Before we start learning about GenServers and Supervisors let's try to figure out what does OTP mean.

OTP stands for Open Telecom Platform.
It is a framework and a bundle of tools and libraries written in Erlang.
Even though that acronym has the "Telecom" in it, nowadays the framework is not telecom specific anymore.
It is a general framework for building scalable and distributed systems.
Basically, it helps to solve a wide range of problems you probably need to solve.


## What is a GenServer

Sometimes you don't want to write low-level code to work with the processes.
At the same time, you may want to have more control comparing to Agents and Tasks.
The OTP has an answer to that in the face of GenServers.

A GenServer is a behavior module which allows you to extend your modules to build server side of the client-server implementation.

Everything works in the similar way as it works in previous examples. We start a new process, we send messages to it, the process listens to the messages and responds back.

Let's start with the example and envolve it step by step.

As usual, I will take ToyRobot implementation and use it in the examples below.
You can find more about it in [the previous articles](http://whatdidilearn.info/tags#Toy-Robot) or grab the code from the [GitHub project page](https://github.com/ck3g/toy_robot).

Let's create the new module with the following content:

```elixir
defmodule ToyRobot.OtpRobot do
  use GenServer

  def handle_call(:report, _from, current_state) do
    report = ToyRobot.report(current_state)
    {:reply, report, current_state}
  end
end
```

And review what do we have here.

Once you start using the `GenServer` module (`use GenServer` line) it defines set of callbacks with the required behavior.
We need to override these callbacks in case we want to use them. That exactly what we did with the `handle_call` callback.
Becuase next thing that I would like to try is to send a message to our process.

You can find the complete list of the available callbacks [on the documentation page](https://hexdocs.pm/elixir/GenServer.html#callbacks).
Here we will cover several of them.

Every callback has its own set of arguments and returns value structure which we need to follow.
In most cases, a return value is a tuple with status as a first element and additional information for the following elements.
In case of `handle_call` it expects three arguments from the GenServer:

* `:report` - is message identifier
* `_from` - is the caller (PID) of the message
* `current_state` - is the current state

and returns tuple of three elements:

* `:reply` - The result of the call
* `report` - The result we will get back
* `current_state` - The next state. In our case, `ToyRobot.report` does not change the state of the robot, so it remains the same

Now let's fire up Interactive Elixir and try to send a message to the server.

```elixir
iex> {:ok, state} = ToyRobot.place
{:ok, %ToyRobot.Position{facing: :north, x: 0, y: 0}}

iex> {:ok, pid} = GenServer.start_link(ToyRobot.OtpRobot, state)
{:ok, #PID<0.161.0>}

iex> GenServer.call(pid, :report)
{0, 0, :north}
```

At first, we place our robot and capture its state.
Then we start our GenServer by providing the module name and the initial state.
We got the process identifier back.
Then we use that identifier to send a `:report` message to GenServer. We do it using [`GenServer.call/3`](https://hexdocs.pm/elixir/GenServer.html#call/3) function.
The `call` function makes a call to the server and waits for the response.
That is where our callback function is being invoked and we got the result back.

Not every time you would need to get a response back from the server.
Sometimes it would be enough to send a message and forget about that.
The movement functionality of the toy robot falls into that category.

To achive that, GenServer provides [`cast/2`](https://hexdocs.pm/elixir/GenServer.html#cast/2) function.

Let's implement the corresponding callback first.

```elixir
def handle_cast(:move, current_state) do
  {:noreply, ToyRobot.move(current_state)}
end
```

This time we respond with `:noreply` as a status and pass an updated state back to GenServer.

We can see it is working in the Interactive Elixir session.

```elixir
iex> GenServer.cast(pid, :move)
:ok

iex> GenServer.call(pid, :report)
{0, 1, :north}
```

The implementation of turning the robot left and right would be very similar to `:move`.

In these examples, we are using GenServer's functions directly and we need to always pass PID.
What if we write a couple of functions to reduce the noise from additional information.

The first thing we need is the start of the process, which is considered with the placing the robot functionality.

```elixir
def place do
  {:ok, state} = ToyRobot.place
  GenServer.start_link(__MODULE__, state, name: __MODULE__)
end
```

All the same, as we did in `iex`. We place the robot, we fire up a process.
We use current module name `__MODULE__` to register a process.
That helps us to use module name instead of PID when we talk to the process.

The rest of the implementation is even easier.

```elixir
def report, do: GenServer.call(__MODULE__, :report)

def move,   do: GenServer.cast(__MODULE__, :move)
def left,   do: GenServer.cast(__MODULE__, :left)
def right,  do: GenServer.cast(__MODULE__, :right)
```

That should be enough to make it work.

```elixir
iex> ToyRobot.OtpRobot.place
iex> ToyRobot.OtpRobot.move
iex> ToyRobot.OtpRobot.left
iex> ToyRobot.OtpRobot.report
{0, 1, :west}
```

And it does. Now we have the robot fully functioning using GenServer inside.
Now we are going to cover a little bit more. We will think how to make it more robust.


## Working with Supervisors

In the [Exceptions in Elixir](http://whatdidilearn.info/2017/11/19/exceptions-in-elixir.html) I've already mentioned,
that Elixir (so does Erlang) endorses "Fail-Fast" philosophy and it is our job to build our applications in the way they can recover themselves from the crashes.

That is cool. But it would be much cooler to learn how exactly can we do that.

Imagine something happens with our robot during its maintenance and you lose the connection with it.
For simplicity, I would add a new function to the ToyRobot module which will trigger a failure.

```elixir
# lib/toy_robot.ex
defmodule ToyRobot do
  # ...
  def failure do
    raise "Connection has been lost"
  end
end
```

Now we need to reach that message from our `ToyRobot.OtpRobot` implementation.

```elixir
# lib/otp_robot.ex
def trigger_failure do
  GenServer.cast(__MODULE__, :failure)
end

def handle_cast(:failure, _current_state) do
  {:noreply, ToyRobot.failure}
end
```

Let's see how will it behave:

```elixir
iex(1)> ToyRobot.OtpRobot.place
{:ok, #PID<0.160.0>}

iex(2)> ToyRobot.OtpRobot.move
:ok

iex(3)> ToyRobot.OtpRobot.report
{0, 1, :north}

iex(4)> ToyRobot.OtpRobot.trigger_failure
:ok
** (EXIT from #PID<0.158.0>) evaluator process exited with reason: an exception was raised:
    ** (RuntimeError) Connection has been lost
        (toy_robot) lib/toy_robot.ex:110: ToyRobot.failure/0
        (toy_robot) lib/otp_robot.ex:37: ToyRobot.OtpRobot.handle_cast/2
        (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
        (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
        (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3

Interactive Elixir (1.5.1) - press Ctrl+C to exit (type h() ENTER for help)

iex(1)>
12:18:39.047 [error] GenServer ToyRobot.OtpRobot terminating
** (RuntimeError) Connection has been lost
    (toy_robot) lib/toy_robot.ex:110: ToyRobot.failure/0
    (toy_robot) lib/otp_robot.ex:37: ToyRobot.OtpRobot.handle_cast/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: {:"$gen_cast", :failure}
State: %ToyRobot.Position{facing: :north, x: 0, y: 1}

nil

iex(2)> ToyRobot.OtpRobot.report
** (exit) exited in: GenServer.call(ToyRobot.OtpRobot, :report, 5000)
    ** (EXIT) no process: the process is not alive or there's no process currently associated with the given name, possibly because its application isn't started
    (elixir) lib/gen_server.ex:766: GenServer.call/3
```

We can see here if something happened with our robot, it dies and we completely lost the connection with it.

Now let's try to fix that using OTP Supervisors.

The Supervisor is also a process which supervises other processes.

Let's create a `ControPanel` module and turn it into supervisor to control our robot.

```elixir
defmodule ToyRobot.ControlPanel do
  use Supervisor

  def deploy_robot do
    Supervisor.start_link(__MODULE__, :ok, [])
  end

  def init(:ok) do
    children = [ToyRobot.OtpRobot]
    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

Using the `Supervisor` module is similar to using the `GenServer`.
In the `deploy_robot` function we are starting a new process.
In the `init` callback, which would be invoked after `start_link/3` call, we initialize the supervisor.
We provide it a list of its children (processes to supervise). In our case, it is `ToyRobot.OtpRobot` module.
And we also specify the supervising strategy to be `:one_for_one`. That means if our supervisor will have more than one children, it will restart only the child process which fails and the rest would not be affected.

Here you can find [the complete list of strategies](https://hexdocs.pm/elixir/Supervisor.html#module-strategies) and description of them.

Once the supervisor is started it will also begin to start its children by calling `start_link` function of that child.
As soon as we don't have it yet for our `OtpRobot`, it is time to define that function.


```elixir
defmodule ToyRobot.OtpRobot do
  # ...
  
  def start_link(_opts) do
    place
  end

  def init(current_state) do
    {:ok, current_state}
  end
end
```

In our case, `start_link` function is exactly the same a `place` function.
All we want is just to place the robot in a default position. We don't care about additional params.
We also define the `init` callback and just pass the current state through.

Now we are ready to see it in action.

```elixir
iex(1)> ToyRobot.ControlPanel.deploy_robot
{:ok, #PID<0.160.0>}

iex(2)> ToyRobot.OtpRobot.move
:ok

iex(3)> ToyRobot.OtpRobot.report
{0, 1, :north}

iex(4)> ToyRobot.OtpRobot.trigger_failure
:ok

iex(5)>
13:18:16.915 [error] GenServer ToyRobot.OtpRobot terminating
** (RuntimeError) Connection has been lost
    (toy_robot) lib/toy_robot.ex:110: ToyRobot.failure/0
    (toy_robot) lib/otp_robot.ex:45: ToyRobot.OtpRobot.handle_cast/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: {:"$gen_cast", :failure}
State: %ToyRobot.Position{facing: :north, x: 0, y: 1}

nil

iex(6)> ToyRobot.OtpRobot.report
{0, 0, :north}
```

The first thing we did is start our supervisor, then we were able to move the robot and check its position.
After that, we send a failure message to it in order our process to fail.
And finally, we check that our robot is available again because our supervisor restarted it.
We also see that the robot is in default position again. That means we lost the data after the failure.

That is something, but still not enough. We need to fix that as well.

In order to recover the state of the robot, we need to keep that state somewhere else. So we can get it from that place.
We can either write a new module and use GenServer functionality there or we can use an [Agent](http://whatdidilearn.info/2017/12/24/agents-and-tasks-in-elixir.html#agents) to kee the state there.
All we need to do is to save the state and read it while our robot starts.


The `GenServer` module provides us the `terminate/2` callback. That callback invoked once the process is about to terminate.
I think that can be a good place where we save the state before we lose it.

```elixir
defmodule ToyRobot.OtpRobot do
  # ...
  
  def terminate(_reason, current_state) do
    Agent.update(:robot_state_repository, fn (_) -> current_state end)
  end
end
```

Here we are updating the state of the agent. We also need to start the agent together with deploying the robot from the control panel. We will update `ToyRobot.ControlPanel.deploy_robot` to do that.

```elixir
def deploy_robot do
  Agent.start_link(fn -> nil end, name: :robot_state_repository)
  Supervisor.start_link(__MODULE__, :ok, [])
end
```

Let's see that in action

```elixir
iex(1)> ToyRobot.ControlPanel.deploy_robot
{:ok, #PID<0.167.0>}

iex(2)> Agent.get(:robot_state_repository, &(&1))
nil

iex(3)> ToyRobot.OtpRobot.move
:ok

iex(4)> ToyRobot.OtpRobot.trigger_failure
:ok

iex(5)>
14:11:03.823 [error] GenServer ToyRobot.OtpRobot terminating
** (RuntimeError) Connection has been lost
    (toy_robot) lib/toy_robot.ex:110: ToyRobot.failure/0
    (toy_robot) lib/otp_robot.ex:45: ToyRobot.OtpRobot.handle_cast/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: {:"$gen_cast", :failure}
State: %ToyRobot.Position{facing: :north, x: 0, y: 1}

nil

iex(6)> Agent.get(:robot_state_repository, &(&1))
%ToyRobot.Position{facing: :north, x: 0, y: 1}

iex(7)> ToyRobot.OtpRobot.report
{0, 0, :north}
```

So we deploy the robot. We check that the agent is up and running as well.
Then we move the robot and trigger the failure.
We see that agent contains the correct state.
Now we need to recover that state together with the recovering of the robot.

We can do that in the `ToyRobot.OtpRobot.init/1` callback:

```elixir
def init(_) do
  current_state =
    case Agent.get(:robot_state_repository, &(&1)) do
      nil ->
        {:ok, state} = ToyRobot.place
        state
      state -> state
    end

  {:ok, current_state}
end
```

If the current state in the Agent is `nil`, that means the control panel started for the first time and we need to place the robot in the default position.
If we have the value inside, then we are recovering the robot.

```elixir
iex(1)> ToyRobot.ControlPanel.deploy_robot
{:ok, #PID<0.162.0>}

iex(2)> ToyRobot.OtpRobot.move
:ok

iex(3)> ToyRobot.OtpRobot.trigger_failure
:ok

iex(4)>
14:37:51.958 [error] GenServer ToyRobot.OtpRobot terminating
** (RuntimeError) Connection has been lost
    (toy_robot) lib/toy_robot.ex:110: ToyRobot.failure/0
    (toy_robot) lib/otp_robot.ex:53: ToyRobot.OtpRobot.handle_cast/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: {:"$gen_cast", :failure}
State: %ToyRobot.Position{facing: :north, x: 0, y: 1}

nil

iex(5)> ToyRobot.OtpRobot.report
{0, 1, :north}
```

Now, every time we reconnect with our robot, it has its previous state.


## Wrapping up

I think we have learned a lot today. Now we know that does OTP means. We can build our own GenServer and we know how to build fault resistant applications using Supervisors. But the journey does not end here. There is way more stuff in OTP.

The complete example fo the code you can find on [the GitHub](https://github.com/ck3g/toy_robot/pull/4) of that project.
