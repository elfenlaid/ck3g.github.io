---
layout: post
title: "Functions in Elixir"
desc: "Learn how to define and use different type of functions in Elixir"
keywords: "Functions, Anonymous functions, Capture functions, Arity, FizzBuzz"
tags: [Elixir]
---

Once you start developing any program besides the very simple one you will absolutely start structuring your code in subroutines.
In the Object Oriented Programming, they called methods. But as soon as Elixir is a functional programming language, subroutines here are functions. Let's take a look what functions can give us.

## Named functions

Most of the time we are using named functions. In Elixir, those functions are live inside modules.

Let's take a look at a simple example.

```elixir
defmodule Greetings do
  def say_hello(name) do
    IO.puts "Hello, #{name}!"
  end
end
```

So here we have a module `Greetings`.
To define modules we are using `defmodule` keyword.
The module consists of `say_hello/1` function who accepts a single argument `name`
and then prints a hello message on the screen.
As you can see functions defined using `def` keyword.


```elixir
iex> Greetings.say_hello("John")
Hello, John!
```

We can also call functions without parentheses because they are optional in Elixir:

```elixir
iex> Greetings.say_hello "John"
Hello, John!
```

### Arity

You are probably noticed that I have referred to a function as a `say_hello/1`.
Ok, we can see that the name of the function is `say_hello`.
But what does `/1` mean?

In Elixir we can define several functions with the same name.
In order to understand that those functions are actually different the arity is used.
Term "Arity" is not Elixir specific. As [Wikipedia](https://en.wikipedia.org/wiki/Arity) says:

> In logic, mathematics, and computer science, the arity of a function or operation is the number of arguments or operands that the function takes.

So we have a function with a single argument, so its arity is 1.

We can define more functions in this module. Let's do this:

```elixir
def say_hello(first_name, last_name) do
  IO.puts "Hello, #{first_name} #{last_name}!"
end

def say_hello(title, first_name, last_name) do
  IO.puts "Hello, #{title} #{first_name} #{last_name}!"
end


iex> Greetings.say_hello("John", "Doe")
Hello, John Doe!

iex> Greetings.say_hello("Mr.", "John", "Doe")
Hello, Mr. John Doe!
```

So now we have `say_hello/1`, `say_hello/2` and `say_hello/3`.

### Default params

Now imagine we are greeting gentlemen most of the time,
but we still want to be polite and refer to them using "Mr." title.

So we can try to make title to have a default value of "Mr.".
To achieve it our attribute in function definition should be followed by `\\ <value>`, which is `\\ "Mr."` in our case.
Let's try to change our function in the following way:

```elixir
def say_hello(title \\ "Mr.", first_name, last_name) do
  IO.puts "Hello, #{title} #{first_name} #{last_name}!"
end
```

But once we try to compile our code we will get:
```
warning: this clause cannot match because a previous clause at line 6 always matches
```

That happens because now Elixir does not know which function to call,
by excluding `title` we have two functions with arity 2.

We may think: "Ok, let's swap the order of the functions and put our `say_hello/3` in front of `say_hello/2` so Elixir can pick up default parameter earlier.
But then we get:

```
** (CompileError) hello.ex:10: def say_hello/2 conflicts with defaults from def say_hello/3
```

So we need to take default parameters into account and check if that does not overlaps with our already existed functions with the same name. 

We have caught ourselves into the trap by trying to set a default value for `title`.

Let's take a look a working example with default parameters to understand how does it work.

```elixir
defmodule DefaultParams do
  def func(a, b \\ "b", c) do
    IO.inspect [a, b, c]
  end
end


iex> DefaultParams.func(1, 2)
[1, "b", 2]

iex> DefaultParams.func(1, 2, 3)
[1, 2, 3]
```

In that example, we have 3 attributes and one of them has a default value.
That means if we call the function with only two attributes Elixir would set values for `a` and `c` param.

### Private functions

By default all the functions defined inside the module are public.
There is a way to define private functions as well. 
Private functions will not be visible from the outside of the module.
To do that we are using `defp` keyword.

### Capture functions

Sometimes we need to store the function into a variable and pass it to another function.
By store function into a variable, I am not talking about the value that function returns,
I am talking about the function itself.
That means we are not executing function body right away, but pass it into another function and then call it inside.

That process is known as function capture.

To do achieve this we can use following structure:

```elixir
variable = &ModuleName.function_name/arity
```

Once we have the function captured we can call it: `

```elixir
variable.(arg1, arg2)
```

Pay attention there is a dot `.` between variable and parentheses.

Let's take a look at the following example to see it in action:

```elixir
defmodule Math do
  def add(a, b) do
    a + b
  end
end

defmodule CaptureExample do
  def do_something(c, d, func) do
    func.(c, d)
  end
end
```

Now we can use our code in the following way:

```elixir
# We can pature the function into variable
iex> sum = &Math.add/2
&Math.add/2

# We can call that function with different arguments
iex> sum.(1, 2)
3

# We can pass variable as argument to another function 
iex> CaptureExample.do_something(1, 2, sum)
3

# Or we can capture and pass it as argument at the same time
iex> CaptureExample.do_something(1, 2, &Math.add/2)
3
```

### Short form

Sometimes a body of our functions can be so small, so we may want to write a complete function definition as a one-liner.
That is also possible in elixir.

Let's rewrite our previous example and make it shorter.

```elixir
defmodule Math do
  def add(a, b), do: a + b
end

defmodule CaptureExample do
  def do_something(c, d, func), do: func.(c, d)
end
```

What we have got here is a function name with arguments followed by comma `,` then beginning of the bloc followed by colon `:` and then function body itself.

I would recommend avoiding using this approach for nontrivial functions.
These functions would harder to read for other people and even for you after some time.

Let's summarise what did we learn here in an example.

### FizzBuzz example

Yes that is solution to [FizzBuzz](http://wiki.c2.com/?FizzBuzzTest) problem without using any conditionals

```elixir
defmodule FizzBuzz do
  def fizz_buzz(n) do
    calculate_fizz_buzz(rem(n, 3), rem(n, 5), n)
  end

  defp calculate_fizz_buzz(0, 0, _), do: "FizzBuzz"
  defp calculate_fizz_buzz(0, _, _), do: "Fizz"
  defp calculate_fizz_buzz(_, 0, _), do: "Buzz"
  defp calculate_fizz_buzz(_, _, n), do: n
end
```

```elixir
→ iex fizz_buzz.ex

iex> Enum.map(1..20, &FizzBuzz.fizz_buzz/1)
[1, 2, "Fizz", 4, "Buzz", "Fizz", 7, 8, "Fizz", "Buzz",
 11, "Fizz", 13, 14,"FizzBuzz", 16, 17, "Fizz", 19, "Buzz"]
```
 
In this example, we have four private `calculate_fizz_buzz/3` functions with the same arity.
As soon as we are using (Pattern Matching](2017/09/21/pattern-matching-in-elixir.html) in the function arguments,
Elixir can understand what function do we want to call.

Pay attention that the order of how do we define these functions are important,
because if we put our `(_, _, n)` function in front of others it would overlap any other function.
 
Then we call that function with `rem(n, 3), rem(n, 5), n` (`rem/2` returns the remainder after dividing one number by another).

In the Interactive Elixir, we use `Enum.map/2` to go from 1 to 20 and use function capture to pass our `fizz_buzz/1` function and see the result.

## Anonymous functions

Sometimes we may need shorter or one-time functions.
We don't even want to bother by organizing them in the modules.
We may need this kind of function just to use once or only to pass as an argument.

So to respond to these needs Elixir provides us Anonymous functions.

To define these functions we are using `fn` keyword using the following format:

```elixir
fn
  arguments -> body
  arguments -> body
end
```

A function which adds two numbers together will look like

```elixir
iex> sum = fn a, b -> a + b end
#Function<12.99386804/2 in :erl_eval.expr/5>

iex> sum.(1, 2)
3
```

We can see that we are calling it using the same way as we did with captured functions.
Well, actually capture is converting named function to anonymous and that is the way how do we call anonymous functions in Elixir.
It worth to mention that parentheses, in this case, are mandatory.

### Multiple bodies

Using Pattern Matching we can have several bodies for functions. For example:

```elixir
iex> sum = fn
...> 0, b -> b
...> a, b -> a + b
...> end
#Function<12.99386804/2 in :erl_eval.expr/5>

iex> sum.(0, 1)
1
iex> sum.(1, 1)
2
```

But we cannot have a different number of arguments within the same function.
So the following code is invalid:

```elixir
iex> sum = fn
...> a, b -> a + b
...> a, b, c -> a + b + c
...> end
** (CompileError) iex: cannot mix clauses with different arities in anonymous functions
```

### More about & notation

The `&` is not only used to capture the function. It can be used to write a shorter version of anonymous functions.

What? Even shorter? Well, Yes.

Now our `sum` function looks like that:

```elixir
# Before
iex> sum = fn a, b -> a + b end
#Function<12.99386804/2 in :erl_eval.expr/5>

# After
iex> sum = &(&1 + &2)
&:erlang.+/2

iex> sum.(1, 2)
3
```

Using `&()` we are defining function body where placeholders `&1` and `&2` are the first and second argument of the function.

Take a look at our updated example from function capture section:

```elixir
iex> CaptureExample.do_something(1, 2, &(&1 + &2))
3
```

### FizzBuzz example

And as an example, there is a FizzBuzz problem solved using anonymous functions.

```elixir
iex> fizz_or_buzz = fn
...>   (0, 0, _) -> "FizzBuzz"
...>   (0, _, _) -> "Fizz"
...>   (_, 0, _) -> "Buzz"
...>   (_, _, n) -> n
...> end
#Function<6.99386804/1 in :erl_eval.expr/5>

iex> fizz_buzz = fn n -> fizz_or_buzz.(rem(n, 3), rem(n, 5), n) end
#Function<6.99386804/1 in :erl_eval.expr/5>

iex> Enum.map(1..20, fizz_buzz)
[1, 2, "Fizz", 4, "Buzz", "Fizz", 7, 8, "Fizz", "Buzz",
 11, "Fizz", 13, 14, "FizzBuzz", 16, 17, "Fizz", 19, "Buzz"]
```

## Summary

As we can see the function is a cornerstone in the Elixir. That is not surprising because Elixir is a functional language.
We have discovered a lot of new things here and laid the solid foundation for next steps.

If you like the post you can support me but share it and you can also subscribe to get more posts like this.

