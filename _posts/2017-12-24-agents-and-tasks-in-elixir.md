---
layout: post
title: "Agents and Tasks in Elixir"
desc: "Here are learning what is Agens and Tasks and how to use them in Elixir"
keywords: "Elixir, processes, agent, tasks, concurrecncy, keeping state"
tags: [Elixir]
---

In the [previous article](http://whatdidilearn.info/2017/12/17/elixir-multiple-processes-basics.html)
we have covered the basics of working with multiple processes.
In order to make it properly work we need to implement several things.
We need recursively listen to messages, we need to handle timeouts and send messages.
That looks like a lot of stuff which can lead to mistakes. To prevent that, Elixir provides us nice wrappers around processes such as Agents and Tasks.
Agents allow us to keep a state and Tasks help us to run processes in parallel.

Let's check what do we have here.

## Agents

As I've already mentioned above, Agents are background processes which help us to maintain a state.

The usage of an agent is pretty straightforward.
We have the ability to spawn an agent with an initial value, update its state and read the state.

Let's get [the StatefulRobot example](http://whatdidilearn.info/2017/12/17/elixir-multiple-processes-basics.html#keeping-a-state) from the previous article and refactor it to use an Agent.

Of course, we need to start from spawning a new process. The `place` function is responsible for that.

```elixir
# Before
def place do
  {:ok, state} = ToyRobot.place
  pid = spawn_link(fn -> listen(state) end)
  Process.register(pid, StatefulRobot)
  pid
end

# After
def place do
  {:ok, state} = ToyRobot.place
  {:ok, pid} = Agent.start(fn -> state end, name: __MODULE__)
  pid
end
```

To start a new process Elixir provides us `Agent.start/2` function.
It accepts an anonymous function as a first argument and bunch of options as a second argument.

The result of the anonymous function would be the initial state of the agent.
In our case that is the initial position of the robot.

The option `:name`allows us to provide the name for a process.
We will be able to use that name without passing a PID from one function call to another.
`__MODULE__` stands for the current module name. In our case, it is a `ToyRobot.AgentRobot`.

We don't need to implement the "listen" functionality because Elixir does it for us inside the Agent.
So we simply remove the following function:

```elixir
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
```

Now we have our process up and running. Waiting for the other commands.

The next thing we are going to update the `report` function.

```elixir
# Before
def report do
  send(StatefulRobot, {:report, self()})

  receive do
    state -> ToyRobot.report(state)
  end
end

# After
def report do
  Agent.get(__MODULE__, &(ToyRobot.report(&1)))
end
```

To read the current state of the agent we can use `Agent.get/2` function.
It accepts a PID or a registered name of the process as a first argument.
That is where the module name from `__MODULE__` comes in handy.

The second argument is an anonymous function, the first argument of which would be a current state.
So we are passing that state to a `ToyRobot.report` function.

In case you are not familiar with the short syntax of anonymous functions the following two expressions are the same:

```elixir
&(ToyRobot.report(&1))

fn state -> ToyRobot.report(state) end
```

The movement and rotate functions are very similar to the report function.
The difference is that we need to update the state instead of reading it.

```elixir
# Before
def move, do: send(StatefulRobot, {:move})
def left, do: send(StatefulRobot, {:left})
def right, do: send(StatefulRobot, {:right})

# After
def move, do: Agent.update(__MODULE__, &(ToyRobot.move(&1)))
def left, do: Agent.update(__MODULE__, &(ToyRobot.left(&1)))
def right, do: Agent.update(__MODULE__, &(ToyRobot.right(&1)))
```

Similar to `Agent.get/2`, `Agent.update/2` accepts a PID or a registered name of the process as a first argument.
An anonymous function as a second. The argument of that anonymous function would be a current state and the result of the function will be a new state of the Agent.

That is pretty much it that we need to change to fully migrate to agents for the robot.

Let's see it in action. 

```
iex> alias ToyRobot.AgentRobot
iex> AgentRobot.place
#PID<0.161.0>
iex> AgentRobot.right
iex> AgentRobot.move
iex> AgentRobot.move
iex> AgentRobot.report
{2, 0, :east}
```

It works!

## A side note

Just a side note about heavy jobs.
Long story short, it is not recommended to use heavy tasks inside an agent.
An agent can be blocked until that task is finished.
So it is always better to perform those calculations outside.

Let's took our `report` function as an example:

```elixir
def report do
  Agent.get(__MODULE__, &(ToyRobot.report(&1)))
end
```

Here the agent will be blocked until the `ToyRobot.report` finishes its work.
What we can do instead, is to read the state and then pass it to `report` function.
In that case, the agent would be unblocked right away and will be ready to accept new messages.

That would be another approach:

```elixir
def report do
  Agent.get(__MODULE__, &(&1)) |> ToyRobot.report
end
```

The complete example of an `AgentRobot` you can find in [the GitHub repository](https://github.com/ck3g/toy_robot/pull/3).

Now we are ready to move to the next topic.

## Tasks

In Elixir, tasks are processes which run in a background.

Imagine you have several jobs you need to do at the same time and the order of those jobs does not matter.
For example, we need to update something in DataBase, send an Email and make a remote call to API.
Those jobs can be somehow heavy and if you want to call them one by one the amount of time to accomplish them,
would be the sum of the time required to accomplish every single job.

Why would we spend so much time then?

With the Elixir tasks, we can run all these jobs in parallel and then collect the result (if we need it).

To start a task we are using `Task.async/1`, which will run the function in the background and send a result of that function back as a payload of the message. Then we use `Task.await` to wait (if needed) and collect the result.

Let's get the following module as an example:

```elixir
# task_example.ex
defmodule TaskExample do
  def db_update do
    "DB update result"
  end

  def send_email do
    {:ok, "Email has been sent"}
  end

  def notify_remote_api do
    {:ok, "Notification has been sent"}
  end
end
```

Then fire up an iex session and play around.

```elixir
iex> c "task_example.ex"
iex> db_query = Task.async(fn -> TaskExample.db_update end)
iex> email = Task.async(fn -> TaskExample.send_email end)
iex> api = Task.async(fn -> TaskExample.notify_remote_api end)
```

At first, we are using `Task.async/1` to create a separate process for every function we have.
Once the process is started it does not block the execution of the next command, thus we can start the next process immediately.

While we are waiting for the completion of the tasks we can do some other stuff as well.

```elixir
iex> IO.puts "Here we can do some other stuff"
Here we can do some other stuff
```

And finally, once we reached the point where we need to know the result of the tasks we can collect it using `Task.await/1`.

```elixir
iex> db_result = Task.await(db_query)
"DB update result"

iex> email_result = Task.await(email)
{:ok, "Email has been sent"}

iex> api_response = Task.await(api)
{:ok, "Notification has been sent"}
```

One you use `Task.async/1` to start a new process, you need to finish it with the `Task.await/1`.
Those two functions work in pairs.

Not every job requires the result back.
For example, we might not care about the result of sending an email and/or notification of a remote API.
In these cases, we want to run those jobs in the separate process and just forget about them.

To do that we can use `Task.start/1` function.

```elixir
iex> Task.start(fn ->
...>   {:ok, response} = TaskExample.notify_remote_api
...>   IO.puts response
...> end)
Notification has been sent
{:ok, #PID<0.128.0>}
```

I use `IO.puts/1` here just to show that the task was executed in the background.


## One more side note

There is one more side note about agents and tasks.

In my examples, I was mostly using functions which accept an anonymous function as a param.
Almost all of these functions (if not all) have another version, which accepts a module name, a function name and params explicitly.

You might think there is no difference and that is just yet another way. But there is at least one difference between those versions of the same function.

If you run those functions on different nodes then functions that accept anonymous function only works if they have the same version of the code between the caller (client) and callee (server). If you develop your apps which will support different versions of the code on different nodes, then it is recommended to pass a module name and a function name as arguments.

## Wrapping up

We have discovered here two more drops from the sea of the existing functionality around processes.
You can see how easy to split the work using tasks and how easy to keep the state using agents.

Although we have covered only the basics here, I encourage you to read the documentation about Agents and Tasks.
You can find more useful functions there which can help you to solve different problems.

As always, there is more interesting stuff exists in Elixir and we will cover it in the next articles.
