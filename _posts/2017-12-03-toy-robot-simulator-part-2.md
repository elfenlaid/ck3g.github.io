---
layout: post
title: "The Toy Robot simulator (Part 2)"
desc: "This time we will implement a Toy Robot simulator in Elixir"
keywords: "Elixir, Toy robot, kata"
tags: [Elixir, Toy-Robot]
feature-img: "img/posts/toy-robot.png"
img-light: true
---

In [the first part](http://whatdidilearn.info/2017/11/26/toy-robot-simulator-part-1.html), we have implemented the placement, rotation and for our toy robot. The robot can also provide us the report about its position. At this part, we will proceed with the implementation. We will implement movement functionality and improve our code a little bit.


## Moving the robot

> - move will move the toy robot one unit forward in the direction it is currently facing.

We will start from the test here. We are going to check if `y` coordinate has been changed after we have been moving the robot while it is facing to the north.

```elixir
test "moving robot up if it is facing to the north" do
  position = ToyRobot.place(0, 0, :north)
             |> ToyRobot.move
             |> ToyRobot.report

  assert position == {0, 1, :north}
end
```

Then we the implementation would be:

```elixir
def move(%ToyRobot.Position{x: _, y: y, facing: :north} = robot) do
  %ToyRobot.Position{robot | y: y + 1}
end
```

In this case, we know we are facing north so we explicitly trying to pattern match it in the arguments. By facing north we also know that the robot can change only Y coordinate. That is why we don't care about X coordinate in this function.

Now we can describe tests and implementation for the three our directions.

```elixir
test "moving robot right if it is facing to the east" do
  position = ToyRobot.place(0, 0, :east)
             |> ToyRobot.move
             |> ToyRobot.report

  assert position == {1, 0, :east}
end

test "moving robot down if it is facing to the south" do
  position = ToyRobot.place(4, 4, :south)
             |> ToyRobot.move
             |> ToyRobot.report

  assert position == {4, 3, :south}
end

test "moving robot left if it is facing to the west" do
  position = ToyRobot.place(4, 4, :west)
             |> ToyRobot.move
             |> ToyRobot.report

  assert position == {3, 4, :west}
end
```

```elixir
def move(%ToyRobot.Position{x: x, y: _, facing: :east} = robot) do
  %ToyRobot.Position{robot | x: x + 1}
end

def move(%ToyRobot.Position{x: _, y: y, facing: :south} = robot) do
  %ToyRobot.Position{robot | y: y - 1}
end

def move(%ToyRobot.Position{x: x, y: _, facing: :west} = robot) do
  %ToyRobot.Position{robot | x: x - 1}
end
```

As soon as we explicitly matching the facing direction in every function, Elixir will properly choose the correct version for us.

That's great. Now our robot can move.

```
→ mix test
Compiling 1 file (.ex)
..........

Finished in 0.06 seconds
10 tests, 0 failures
```


### Moving improvements

At this point, we have implemented all required functionality for the robot. But the thing is we have followed only a happy path. If we try to place our robot on the edge it will move and fall from a table.

```elixir
test "prevent the robot to fall" do
  position = ToyRobot.place(4, 4, :north)
             |> ToyRobot.move
             |> ToyRobot.report

  assert position == {4, 4, :north}
end
```

```
→ mix test
Compiling 1 file (.ex)
...

  1) test prevent the robot to fall (ToyRobotTest)
     test/toy_robot_test.exs:87
     Assertion with == failed
     code:  assert position == {4, 4, :north}
     left:  {4, 5, :north}
     right: {4, 4, :north}
     stacktrace:
       test/toy_robot_test.exs:92: (test)

.......

Finished in 0.07 seconds
11 tests, 1 failure
```

That behavior violates the rules.

> - The toy robot must not fall off the table during movement. This also includes the initial placement of the toy robot.
> - Any move that would cause the robot to fall must be ignored.


We need to fix that.

As one of the solutions, we can try to check the value we are changing the permissible value of the coordinate and choose one that is suitable.

For example, when we are moving north we can get the minimum value between table border and the next coordinate. We need to have table size for that first. Then we compare it using `Enum.min/2`.

```elixir
defmodule ToyRobot do
  @table_top_x 4
  @table_top_y 4

  # ...

  def move(%ToyRobot.Position{x: _, y: y, facing: :north} = robot) do
    %ToyRobot.Position{robot | y: Enum.min([@table_top_y, y + 1])}
  end

  # ...
end

```

Another solution would be to use [Guards](https://hexdocs.pm/elixir/master/guards.html). 
We can use guards to extend the condition of choosing the right function for the job. On top of pattern matched arguments.
It starts with the keyword `when` followed by a condition. It will have the following syntax if we apply it to the function:

```elixir
def function_name(arguments) when condition do
end
```

Now let's see how will it look for our function.

```elixir
def move(%ToyRobot.Position{x: _, y: y, facing: :north} = robot) when y < @table_top_y do
  %ToyRobot.Position{robot | y: y + 1}
end
```

Having a guard here, we need to remember, that if none of the conditions are satisfied we can get an error:

```
** (FunctionClauseError) no function clause matching in ToyRobot.move/1
```

That means we need to provide the fallback function at the very end of our `move` functions. This fallback function will return unchanged state of the robot.

```elixir
def move(robot), do: robot
```

I see both approaches as viable. It's up to you to decide which one to use. In our context, I think I like the guards more. So I will apply it for the rest of the functions.

Which one do you prefer? And why?

### Placing the robot improvements

Currently, it is possible to place our robot into the invalid position. Luckily for us, by adding guards to our functions, the wrongly placed robot cannot move. On the one hand that can also be a valid solution.

Let's try to improve it by the approach which you will see quite often in some implementations.
Once we place our robot, instead of returning its position we will return a tuple of two elements.
The first element will contain the result of the operation: `:ok` for success and `:failure` for the wrong operation.
The second element will contain the position in the successful case and the error message in case of failure.

The same approach is used, for example, to open the file using [File.open/2](https://hexdocs.pm/elixir/master/File.html#open/2).

Let's change our tests for `ToyRobot.place` functions to "expect" these changes.

By the way, we would need to update most of the tests where we are using `place` functions. The complete result you can find on the [GitHub repository of that solution](https://github.com/ck3g/toy_robot/tree/v1.0). That is how it would look for "rotate our robot" test:

```elixir
test "rotates the robot to the left" do
  {:ok, robot} = ToyRobot.place(0, 0, :north)
  position = robot |> ToyRobot.left |> ToyRobot.report

  assert position == {0, 0, :west}
end
```

Now back to our `place` functions.

```elixir
test "does not place the robot outside of the table" do
  assert ToyRobot.place(-1, -1, :north) == {:failure, "Invalid position"}
end
```

```
→ mix test
...........

  1) test does not place the robot outside of the table (ToyRobotTest)
     test/toy_robot_test.exs:13
     Assertion with == failed
     code:  assert ToyRobot.place(-1, -1, :north) == {:failure, "Invalid position"}
     left:  {:ok, %ToyRobot.Position{facing: :north, x: -1, y: -1}}
     right: {:failure, "Invalid position"}
     stacktrace:
       test/toy_robot_test.exs:14: (test)

...

Finished in 0.09 seconds
15 tests, 1 failure
```

Do you still remember about guards? Great. Because they can help us here as well.

```elixir
def place(x, y, _facing)
when x < 0 or y < 0 or x > @table_top_y or y > @table_top_y
do
  {:failure, "Invalid position"}
end
```

Ok. The last thing to make our placing functions more stable is to prevent invalid facing direction.
Similiar to coordinates, we need a test and an implementation.

```elixir
test "does not place the robot with invalid facing direction" do
  assert ToyRobot.place(0, 0, :north_west) == {:failure, "Invalid facing direction"}
end
```

```elixir
def place(_x, _y, facing)
when facing not in [:north, :east, :south, :west]
do
  {:failure, "Invalid facing direction"}
end
```

Now our robot is much much stable than before.

### Replace some tests with doctest

Ok. That is the last thing for now. I promise.
I would like to replace some tests with doctests. (In case you don't know what doctests are you can read about them in the [Writing documentation](http://whatdidilearn.info/2017/10/23/writing-documentation-in-elixir.html) article.)

```elixir
@doc """
Places the robot in the default position

Examples:

    iex> ToyRobot.place
    {:ok, %ToyRobot.Position{facing: :north, x: 0, y: 0}}
"""
def place do
  {:ok, %ToyRobot.Position{}}
end
```

And now we can get rid of:

```elixir
test "places the Toy Robot on the table in the default position" do
```

Next is:

```elixir
@doc """
Places the robot in the provided position,
but prevents it to be placed outside of the table and facing invalid direction.

Examples:

    iex> ToyRobot.place(1, 2, :south)
    {:ok, %ToyRobot.Position{facing: :south, x: 1, y: 2}}

    iex> ToyRobot.place(-1, -1, :north)
    {:failure, "Invalid position"}

    iex> ToyRobot.place(0, 0, :north_east)
    {:failure, "Invalid facing direction"}
"""
def place(x, y, facing) do
  {:ok, %ToyRobot.Position{x: x, y: y, facing: facing}}
end
```

And so on and so forth. Do it as much I as you can proceed. The final version you find in [this commit](https://github.com/ck3g/toy_robot/commit/92b4e23c881f9f8341882a8378f5e2b03faeb17a).


## Wrapping up

Experienced Elixir developers can probably spot mistakes I've made here. Either it is a functional programming mistake or non-Elixir-way mistakes. It is fine for now. But if you can point me to some improvements I would appreciate that.

Besides that, this Kata is implemented in a sense, well, at least it working. Even if that not a perfect Elixir/functional way.
Even by implementing that task we have covered bunch of Elixir topics. If you are new to Elixir and feel a lack of understanding of some parts, take a look at some articles I already have here. Most likely you will find answers to your questions. If not, let me know, I will try to shed the light on these topics.

So, we are done by now. But more to come.
