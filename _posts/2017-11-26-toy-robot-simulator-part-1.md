---
layout: post
title: "The Toy Robot simulator (Part 1)"
desc: "This time we will implement a Toy Robot simulator in Elixir"
keywords: "Elixir, Toy robot, kata"
tags: [Elixir, Toy-Robot]
feature-img: "img/posts/toy-robot.png"
img-light: true
---

Now as we already covered the basics of Elixir in the previous articles we can try to practice a little bit.
Here I would like to solve the Toy Robot Simulator problem which I've
found [here](https://github.com/fredwu/toy-robot-elixir/blob/master/PROBLEM.md).
I guess that is not original source and it was reposted at many other resources.

Fast forward a little bit. As soon my brain thinks in terms of Object-Oriented Programming (yet), my solution, most likely,
would not be perfect from the functional programming point of view.
That is fine for now. That is the point. Because I want to practice and will try to apply practices of functional programming as much as I can.

I will try to provide links to previous articles so you can read more about certain topics once you feel lack of knowledge in that area.

So let's start by creating a new application. (More about using mix to create applications [here](http://whatdidilearn.info/2017/09/14/elixir-first-steps.html#what-is-mix))

```
→ mix new toy_robot
```

I've also cleaned up the boilerplate code a little bit.
Let's look at the requirements again:

> Create an application that can read in commands of the following form:
>
>    PLACE X,Y,F<br />
>    MOVE<br />
>    LEFT<br />
>    RIGHT<br />
>    REPORT<br />

We can start by implementing our `place` function.

## Place the robot into initial position

> - PLACE will put the toy robot on the table in position X,Y and facing NORTH, SOUTH, EAST or WEST.


Let's start with the unit test.

```elixir
defmodule ToyRobotTest do
  use ExUnit.Case
  doctest ToyRobot

  test "places the Toy Robot on the table in the default position" do
    assert ToyRobot.place == %ToyRobot.Position{x: 0, y: 0, facing: :north}
  end
end
```

I am describing the `ToyRobot.place/0` function here which will be used to place the robot in the default position.
As soon as we need to operate with a position of the Robot, I find the Structure to be working well for that. (More about structures [here](http://whatdidilearn.info/2017/11/06/more-on-maps-and-structs-in-elixir.html#structs))

Then we will run our tests, check the error and proceed the iteration.

```
→ mix test
** (CompileError) test/toy_robot_test.exs:6: ToyRobot.Position.__struct__/1 is undefined, cannot expand struct ToyRobot.Position
```

Let's create new file `lib/position.ex` there we will define our structure:

```elixir
defmodule ToyRobot.Position do
  defstruct x: 0, y: 0, facing: :north
end
```

Run the tests again:

```
→ mix test
Compiling 1 file (.ex)
Generated toy_robot app


  1) test places the Toy Robot on the table in the default position (ToyRobotTest)
     test/toy_robot_test.exs:5
     ** (UndefinedFunctionError) function ToyRobot.place/0 is undefined or private
     code: assert ToyRobot.place == %ToyRobot.Position{x: 0, y: 0, facing: :north}
     stacktrace:
       (toy_robot) ToyRobot.place()
       test/toy_robot_test.exs:6: (test)



Finished in 0.04 seconds
1 test, 1 failure
```

Now we need to define our `ToyRobot.place/0` function.

```elixir
defmodule ToyRobot do
  def place do
    %ToyRobot.Position{}
  end
end
```

And we are green now.

```
→ mix test
Compiling 1 file (.ex)
.

Finished in 0.03 seconds
1 test, 0 failures
```

As the next step, we will proceed with the implementation of `ToyRobot.place/3`. First goes the test:

```elixir
test "places the Toy Robot on the table in the specified position" do
  assert ToyRobot.place(1, 2, :south) == %ToyRobot.Position{x: 1, y: 2, facing: :south}
end
```

and the implementation itself

```elixir
def place(x, y, facing) do
  %ToyRobot.Position{x: x, y: y, facing: facing}
end
```

In this case, we place our robot into coordinates provides to us.


## Get reports from the Robot

Once we have our robot at the position, we can start asking it to provide a report to us.
Even if we know his position anyway, this functionality would be still useful for us.
It would be easy for us to check the following functions.

Quick glance at the requirements

> - REPORT will announce the X,Y and F of the robot. This can be in any form, but standard output is sufficient.

I think we can use tuple as a result in the following format `{x, y, facing}`. We will start with the test for that:

```elixir
test "provides the report of the robot's position" do
  robot = ToyRobot.place(2, 3, :west)
  assert ToyRobot.report(robot) == {2, 3, :west}
end
```

The implementation is pretty simple.
All we need to do is to fetch those values from the robot's position structure.
We can achieve using the power of the [Pattern Matching](http://whatdidilearn.info/2017/09/21/pattern-matching-in-elixir.html).

```elixir
def report(robot) do
  %ToyRobot.Position{x: x, y: y, facing: facing} = robot

  {x, y, facing}
end
```

In this implementation, we are using function attribute on the right side and we are matching it with the structure on the left side.
It will assign required values to our `x`, `y` and `facing` variables, which we use to build a tuple.

Now as soon as we are using the `robot` argument only for extract values from it, we can move pattern matching expression right into function params.
We can also ignore the value itself by renaming it into `_robot`.

```elixir
def report(%ToyRobot.Position{x: x, y: y, facing: facing} = _robot) do
  {x, y, facing}
end
```

That is the complete implementation of `ToyRobot.report/1`.

If you came from the Object-Oriented background you can notice the difference in the implementation here.
In OOP we usually have an object, objects know their state. In our case, it is position of the robot. Then we send a message `report` to an object without additional arguments and the object itself can provide all required data.

Well, actually it should be separate `Report` class which receives the object and build the report. The object itself should not know how to report itself. But anyway, those are details of the implementation. I just want to make it as simple as possible.

So, unlike OOP in functional programming, we need to pass all required data to the function as arguments. What is why our report function expects the following structure.


## Rotate the robot

Now it's time to implement rotation of the robot.

> - LEFT and RIGHT will rotate the robot 90 degrees in the specified direction without changing the position of the robot.

First goes the test:

```elixir
test "rotates the robot to the right" do
  position = ToyRobot.place(0, 0, :north)
             |> ToyRobot.right
             |> ToyRobot.report

  assert position == {0, 0, :east}
end
```

Here we have placed our robot into 0x0 position facing to the north, then we rotate it to the right and ask to report the position.
We were using the Pipe Operator here to chain our functions.

#### The Pipe Operator |>

This expression can be written in a less nicer form:

```elixir
robot = ToyRobot.place(0, 0, :north)
robot = ToyRobot.right(robot)
position = ToyRobot.report(robot)
```

The alternative way would be to write it without additional variables:

```elixir
position = ToyRobot.report(ToyRobot.right(ToyRobot.place(0, 0, :north)))
```

The disadvantages of this line are that you need to read it inside out to see the order of function calls.

Elixir provides us a better way to do that. The |> operator takes the result of the function on the left side and passes it as a first argument of the function on its right side.
If a function has more than one argument, you need to pass them except the first one.

For example next two examples are equivalent:

```elixir
function(value1, value2)

value1 |> function(value2)
```

Ok. Now when we become familiar with the Pipe Operator we can proceed.

In the end, we are expecting the robot to remain in the same place, but facing to the east.

To rotate the robot right, we need to know the next facing direction regarding its current direction.
If the robot looks to the north, it should turn to the east.
If it looks to the east, it should turn to the south and so on.

For that case, I think a Map with the all available directions should work well for us. Let's check the implementation of the function.

```elixir
def right(%ToyRobot.Position{facing: facing} = robot) do
  directions_to_the_right = %{north: :east, east: :south, south: :west, west: :north}

  %ToyRobot.Position{robot | facing: directions_to_the_right[facing]}
end
```

Here we have the Map with the directions, passing the current facing direction as a key we can get the next direction.
Then we return the position of the robot containing updated `:facing` value, but keep rest of the attributes the same as before.

Let's extend our test to check more rotation possibilities. For example, if we rotate the robot twice, it should look to the south.

```elixir
test "rotates the robot to the right" do
  position = ToyRobot.place(0, 0, :north)
             |> ToyRobot.right
             |> ToyRobot.report

  assert position == {0, 0, :east}

  position = ToyRobot.place(0, 0, :north)
             |> ToyRobot.right
             |> ToyRobot.right
             |> ToyRobot.report

  assert position == {0, 0, :south}
end
```

Usually, it's considered as a bad practice to have more than one asserts per unit test, but for the sake of example let's keep it like that for now.

Now we need to implement the rotation to the left, which would be similar to the rotation to the right.

```elixir
test "rotates the robot to the left" do
  position = ToyRobot.place(0, 0, :north)
             |> ToyRobot.left
             |> ToyRobot.report

  assert position == {0, 0, :west}
end
```

To turn the robot left we can use the similar Map. It is similar because the rotation direction is different now.
Looking to the north and rotate left, we need to turn the robot to the west. Then to the south and so on.

```elixir
def left(%ToyRobot.Position{facing: facing} = robot) do
  directions_to_the_left = %{north: :west, west: :south, south: :east, east: :north}
  %ToyRobot.Position{robot | facing: directions_to_the_left[facing]}
end
```

That works.

But let's think for a minute, what can we improve here? What the difference between `directions_to_the_right` and `directions_to_the_left`?

The difference is, that is one the inverse version of another. So if we take `directions_to_the_right` and swap the keys and the values, we will get the same map as `directions_to_the_left`.

```elixir
iex>  directions_to_the_right = %{north: :east, east: :south, south: :west, west: :north}
%{east: :south, north: :east, south: :west, west: :north}

iex> Enum.map(directions_to_the_right, fn {from, to} -> {to, from} end)
[south: :east, east: :north, west: :south, north: :west]
```

Let's apply this knowledge to our code. At first, to have access to `directions_to_the_right` let's extract it into [Module attribute](http://whatdidilearn.info/2017/10/16/modules-in-elixir.html#module-attributes) and refactor our `ToyRobot.right/1` function:

```elixir
@directions_to_the_right %{north: :east, east: :south, south: :west, west: :north}
def right(%ToyRobot.Position{facing: facing} = robot) do
  %ToyRobot.Position{robot | facing: @directions_to_the_right[facing]}
end
```

Now we are ready to refactor `ToyRobot.left/1` function as well:

```elixir
@directions_to_the_left Enum.map(@directions_to_the_right, fn {from, to} -> {to, from} end)
def left(%ToyRobot.Position{facing: facing} = robot) do
  %ToyRobot.Position{robot | facing: @directions_to_the_left[facing]}
end
```

We are enumerating through our map and swap the values using [anonymous function](http://whatdidilearn.info/2017/10/06/functions-in-elixir.html#anonymous-functions).

Great. I would like to write one more test just to overlap the checks of the rotation. The idea is to check that rotation thrice into one direction is the same as rotation once in the opposite.

```elixir
test "rotating the robot 3 times to the right is the same as rotating it to the left" do
  right_position = ToyRobot.place(0, 0, :north)
                   |> ToyRobot.right
                   |> ToyRobot.right
                   |> ToyRobot.right
                   |> ToyRobot.report

  left_position =  ToyRobot.place(0, 0, :north)
                   |> ToyRobot.left
                   |> ToyRobot.report

  assert right_position == left_position
end
```

And it also works

```
→ mix test
Compiling 1 file (.ex)
......

Finished in 0.05 seconds
6 tests, 0 failures
```


### Wrapping up the first part

In this part, we have implemented three parts of robots functionality. We are able to place our robot in the specified position, we can rotate it left and right and we can ask robot to report his current position.

See you soon in the second part where we will implement the movement of the robot and apply some improvements for our functions.
