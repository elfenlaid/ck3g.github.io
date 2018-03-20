---
layout: post
title: "Writing a command line app in Elixir"
desc: "Use toy robot to build a command line application in Elixir"
keywords: "Elixir, CLI, Command Line App, Command line, Toy robot, kata"
tags: [Elixir, Toy-Robot]
---

In the previous articles, we have implemented the Toy Robot (you can find it here [Part 1](http://whatdidilearn.info/2017/11/26/toy-robot-simulator-part-1.html) and [Part 2](http://whatdidilearn.info/2017/12/03/toy-robot-simulator-part-2.html)).
This time we will improve the implementation and turn it into a console application.
In that application, we will be able to run the simulator and give commands to the robot.

If you would like to follow these steps on your own, you can use [the code](https://github.com/ck3g/toy_robot/tree/v1.0) as a starting point. Be sure to use v1.0 tag for that.

## Start and welcome

Let's implement the first step of the application. We will be able to run it and display a welcome message.

To turn the app into executable we can use Erlang's `escript` command, which [is supported by Elixir](https://hexdocs.pm/mix/Mix.Tasks.Escript.Build.html) as well.
Before we can "build" that executable we need to describe which module will we use as a starting point for our application.
To do that we need to extend `project` function from `mix.exs` file by adding `escript` option:

```elixir
defmodule ToyRobot.Mixfile do
  use Mix.Project

  def project do
    [
      app: :toy_robot,
      # ...
      escript: escript() # <- That is a new line
    ]
  end

  # ...

  defp escript do
    [main_module: ToyRobot.CLI]
  end
end
```

Now we need to define `ToyRobot.CLI` module. The module should contain `main/1` function, which will be the starting point of the application. For now, let's display our welcome message and quit.

```elixir
defmodule ToyRobot.CLI do
  def main(_args) do
    IO.puts("Welcome to the Toy Robot simulator!")
  end
end
```

We are ready to build the application and use it from a command line. We will use `mix escript.build` to do that.

```
→ mix escript.build
Compiling 3 files (.ex)
Generated toy_robot app
Generated escript toy_robot with MIX_ENV=dev
```

The build command will use the name of the application as a file name. Now we can run it.

```
→ ./toy_robot
Welcome to the Toy Robot simulator!
```

We can also copy that script and run it on any other machine which has Erlang installed.

```
→ cp toy_robot ~/Desktop/
→ ~/Desktop/toy_robot
Welcome to the Toy Robot simulator!
```

Congratulations! First steps are done. We can run the app from the terminal.
But we don't want it to only write a message and quit, do we?
Instead, we want to give commands to our robot.

Let's try to implement our first command.

## The quit command

It would like to start from the simplest command. First, we need a plan that we need to do in that section.
We want our application to display a welcome message followed by help instructions.
If a new user starts using our simulator it would be nice to explain to him what commands he can run.

Next, we need the program to listen to the commands and wait for user's input.

If user types "quit" we want to close the program.
We also want to display an error message in case a command does not support the simulator.

Let's see the extended version of the code and slowly walk through it.

```elixir
defmodule ToyRobot.CLI do
  def main(_args) do
    IO.puts("Welcome to the Toy Robot simulator!")
    print_help_message()
    receive_command()
  end

  @commands %{
    "quit" => "Quits the simulator"
  }

  defp receive_command do
    IO.gets("\n> ")
    |> String.trim
    |> String.downcase
    |> execute_command
  end

  defp execute_command("quit") do
    IO.puts "\nConnection lost"
  end

  defp execute_command(_unknown) do
    IO.puts("\nInvalid command. I don't know what to do.")
    print_help_message()

    receive_command()
  end

  defp print_help_message do
    IO.puts("\nThe simulator supports following commands:\n")
    @commands
    |> Enum.map(fn({command, description}) -> IO.puts("  #{command} - #{description}") end)
  end
end
```

At first, we have extended our `main` function to display the help message right after welcome message.
The help message provides the list of all available commands user can use.
At that moment there is only "quit" command.

Right after display of welcome and help messages we are triggering the `receive_command` function.
This function listens for user's input. Once we get the input we cut the line breaks "\n" and transform it into a lowercase string.
After those transformations, we pass it to an `execute_command` function.

If a user provides us "quit" string we display a farewell message and quit. That means the `defp execute_command("quit")` function is called.

For any other input, we will display an invalid command message followed by the help. We also call the `receive_command` function again, so a user can repeat the process.


## The placement command

Once we finished with the backbone of simulator's interface we can proceed and implement the placement command.

First, let's add it to the list of supported commands.

```elixir
@commands %{
  "quit" => "Quits the simulator",
  "place" => "format: \"place [X,Y,F]\". " <>
             "Places the Robot into X,Y facing F (Default is 0,0,North). " <>
             "Where facing is: north, west, south or east."
}
```

Then we need to implement `execute_command` function for the default case.
If we pass no arguments to `ToyRobot.place` function it would place it into a default position.

```elixir
defp execute_command("place") do
  ToyRobot.place
  receive_command
end
```

Let's check it

```
→ mix escript.build && ./toy_robot
Compiling 1 file (.ex)
Generated escript toy_robot with MIX_ENV=dev
Welcome to the Toy Robot simulator!

The simulator supports following commands:

  place - format: "place [X,Y,F]". Places the Robot into X,Y facing F (Default is 0,0,North). Where facing is: north, west, south or east.
  quit - Quits the simulator

> place
> quit

Connection lost
```

Great. Now we need our `place` command to support arguments.
In that case, we need to change the way of calling our `execute_command` functions.
At first, we need to split the command itself and its attributes.
We can do it by adding `String.split(" ")` step into `receive_command`.

```elixir
defp receive_command do
  IO.gets("> ")
  |> String.trim
  |> String.downcase
  |> String.split(" ") # <- additional step here
  |> execute_command
end
```

Now instead of a string, the `execute_command` function will receive a list.
The first element of that list would contain the command and the second - raw parameters if there are any.
That means we need to update rest of our functions to support that.

```elixir
defp execute_command(["place"])

defp execute_command(["quit"])
```

Now, go build and run the app to check if it still works.
After that, we are ready to proceed with the implementation.

```elixir
defp execute_command(["place" | params]) do
  {x, y, facing} = process_place_params(params)

  case ToyRobot.place(x, y, facing) do
    {:ok, _robot} ->
      receive_command()
    {:failure, message} ->
      IO.puts message
      receive_command()
  end
end
  
defp process_place_params(params) do
  [x, y, facing] = params |> Enum.join("") |> String.split(",") |> Enum.map(&String.trim/1)
  {String.to_integer(x), String.to_integer(y), String.to_atom(facing)}
end
```

This function will be called if a user enters `place` command with arguments.
We checking it by explicitly comparing with `place` command and capture the params.

The `ToyRobot.place/3` function expects coordinates `x` and `y` to be a type of Integer and `facing` as an Atom.
That means we need to transform our string of parameters into those values.
We do that using `process_place_params/1` function.
We are joining the params just in case there is more than one element in the list.
Then we split the string by commas.
Then we cut the garbage out.
Then we turn it into a tuple with valid data.

Once we get our params parsed. We call `ToyRobot.place/3` and passing those params into it.
If that call was not successful we would display the error message.
Either way, we listen to a next user's command.


### The Report command

As usual, let's begin by adding `report` to the list of available commands.

```elixir
@commands %{
  # ...
  "report" => "The Toy Robot reports about its position"
}
```

In order to get reports from the robot, we need to call `ToyRobot.report/1` function.
This function requires us to pass robot's position. Which we didn't know yet.
We are placing our robot, but immediately throw away all the knowledge about its position.

Let's fix that. Once we receive the position of the robot from the `place` command we will pass it through every time.
In that case, we will keep the position of the robot.

```elixir
defp receive_command(robot \\ nil) do
  IO.gets("> ")
  |> String.trim
  |> String.downcase
  |> String.split(" ")
  |> execute_command(robot)
end
```

Our `receive_command` receives the robot as an argument and passes it to the `execute_command` function.
The `robot` is `nil` by default because it was not placed yet.

Let's update all the versions of our `execute_command` functions to receive `robot` argument, and pass it through if needed.

```elixir
defp execute_command(["place"], _robot) do

defp execute_command(["place" | params], _robot) do

defp execute_command(["quit"], _robot) do

defp execute_command(_unknown, robot) do
```

What kind of report would we get if we ask non-placed robot? I think we would respond with an error.

```elixir
defp execute_command(["report"], nil) do
  IO.puts "The robot has not been placed yet."
  receive_command()
end
```

In case the robot is in position, we can display its coordinates:

```elixir
defp execute_command(["report"], robot) do
  {x, y, facing} = robot |> ToyRobot.report
  IO.puts String.upcase("#{x},#{y},#{facing}")

  receive_command(robot)
end
```

```
> place 1, 2, south
> report
1,2,SOUTH
```

Cool. The major part is done. 

### Rotate and move

Now our code is well shaped and we can easily extend it with rotate and move functionality.

As usual, we need to define it as an available command by extending `@commands` attribute.
We will add `left` command:

```elixir
@commands %{
  # ...
  "left" => "Rotates the robot to the left"
}
```

```elixir
defp execute_command(["left"], robot) do
  robot |> ToyRobot.left |> receive_command
end
```

We are receiving a position of the robot, we rotate the robot and pass it through to listen to the following commands.

The result is:

```
> place
> report
0,0,NORTH
> left
> report
0,0,WEST
```

Implementation of the rotation to the right and movement would be pretty similar:

```elixir
@commands %{
  # ...
  "right" => "Rotates the robot to the right",
  "move"  => "Moves the robot one position forward"
}
```

```elixir
defp execute_command(["right"], robot) do
  robot |> ToyRobot.right |> receive_command
end

defp execute_command(["move"], robot) do
  robot |> ToyRobot.move |> receive_command
end
```

```
> place
> report
0,0,NORTH
> move
> right
> move
> left
> move
> report
1,2,NORTH
```

That is it. We have made available all robot's functionality in our command line application.

There is one more thing left.
The subject of writing command line application would not be complete without mention how to work with parameters.
Let's cover that topic.

## Accepting parameters

Our `main/1` function receives attribute. Which we were not using until now.
This attribute contains the list of parameters if they were passed.
It would contain an empty list `[]` if there are no parameters passed.

Let's extend the program to provide a help message if a user runs our program with `--help` attribute like this:

```
./toy_robot --help
```

In that case, our `args` would hold a `["--help"]` value. One way can be just to fetch the `"--help"` value our of the list using pattern matching.
But Elixir ships with the [`OptionParser.parse/2`](https://hexdocs.pm/elixir/OptionParser.html#parse/2) which can help us to parse params.

Let's fire an `iex` and play with it a little bit.
If we run a console app with those parameters:

```
./toy_robot --help instructions.csv --something=else
```

The `args` would contain following list

```
["--help", "instructions.csv", "--something=else"]
```

By passing it to the `OptionParser.parse/2` we would get:

```elixir
> OptionParser.parse(["--help", "instructions.csv", "--something=else"])
{[help: "instructions.csv"], [], [{"--something", "else"}]}
```

The result contains a tuple with 3 lists in it.
The first list is a "parsed" arguments, second is the rest of arguments and the third is a list of invalid arguments.

That result does not look how we want it. We can pass additional parsing options as the second argument to an `OptionParser.parse/2` function. Those options are some kind of rules.

For example. We understand that `--help` option is a boolean, it was either requested or not. We can specify it as following:

```elixir
> OptionParser.parse(["--help", "instructions.csv", "--something=else"], switches: [help: :boolean])
{[help: true], ["instructions.csv"], [{"--something", "else"}]}
```

Now we get it as a boolean. 
The "instructions.csv" now is in the rest parameters, but "something" is still in the invalid. We can fix that as well.

```elixir
> OptionParser.parse(["--help", "instructions.csv", "--something=else"], switches: [help: :boolean, something: :string])
{[help: true, something: "else"], ["instructions.csv"], []}
```

Now it's properly parsed.

You can find a way more options in the [documentation of the `OptionParser.parse/2`](https://hexdocs.pm/elixir/OptionParser.html#parse/2). Now let's proceed and implement our help functionality.

So we only care about `--help` option to display a help message once it exists. Otherwise, we launch the simulator.

```elixir
def main(args) do
  args |> parse_args |> process_args
end

def parse_args(args) do
  {params, _, _} =  OptionParser.parse(args, switches: [help: :boolean])
  params
end

def process_args([help: true]) do
  print_help_message()
end

def process_args(_) do
  IO.puts("Welcome to the Toy Robot simulator!")
  print_help_message()
  receive_command()
end
```

We have changed our `main/1` function. We parse the arguments. We transform them into the format we want.
We have now two versions of `process_args/2` function which we use to either show a welcome message or proceed and start the simulator.

Go ahead, build the app and run `./toy_robot --help` command. You will see a help message and the program will be terminated.

## Wrapping Up

Let's take a quick look back and see what did we achieve here.

Now we know how to build a console application and how to work with command line arguments. On top of that, we have extended the Toy Robot simulator which we can play with it in the interactive mode.

The complete list of the changes you can find on a [GitHub page](https://github.com/ck3g/toy_robot/pull/1) of the project.

See you next time, where we will implement new stuff.
Don't wanna miss a thing? Then subscribe in the form below.
