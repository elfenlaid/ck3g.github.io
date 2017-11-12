---
layout: post
title: "Control flow in Elixir"
desc: "Learn how to control the flow in Elixir. Using conditions, loops and comprehensions"
keywords: "Elixir, conditions, loops, comprehensions, if, cond, case"
tags: [Elixir]
---

Elixir, as a functional programming language, normally follows declarative programming paradigm instead of imperative.
By defining lots of small independent functions and use some tools Elixir provides us, we will use less of control flow constructions comparing to other languages. Well, maybe comparing to imperative languages.

However, we have tools to control a flow in Elixir and it is always good to know them.

## Conditions

Let's start with the most common one.

### The `if`

Elixir provides us the `if/2` macros which receive condition as a first argument and the keyword list as a second:

```elixir
iex> if 1 < 2, do: "Yes. 1 is less than 2"
"Yes. 1 is less than 2"
```

We can also store a result of the expression into a variable:

```
iex> result = if 1 < 2, do: "Yes. 1 is less than 2"
"Yes. 1 is less than 2"

iex> result
"Yes. 1 is less than 2"
```

it supports `else` argument as well.

```elixir
iex> if 1 > 2, do: "Yes. 1 is less than 2", else: "No. It is not"
"No. It is not"
```


Some people might argue it is a ternary operator in Elixir. I'm not sure about that. Even though it is a one-liner, I would barely call it ternary.

Elixir also provides us a version sprinkled with a syntactic sugar:

```elixir
iex> if 1 < 2 do
...>   "Yes. 1 is less than 2"
...> end
"Yes. 1 is less than 2"

iex> if 1 > 2 do
...>   "Yes. 1 is less than 2"
...> else
...>   "No. It is not"
...> end
"No. It is not"
```

There is also a `unless/2` macro, which stands for "if not":

```elixir
iex> unless 1 > 2, do: "Yes. 1 is less than 2"
"Yes. 1 is less than 2"

iex> unless 1 < 2, do: "Yes. 1 is less than 2", else: "No. It is not"
"No. It is not"

iex> unless 1 > 2 do
...>   "Yes. 1 is less than 2"
...> end
"Yes. 1 is less than 2"

iex> unless 1 < 2 do
...>   "Yes. 1 is less than 2"
...> else
...>   "No. It is not"
...> end
"No. It is not"
```

I didn't find the way to implement "if/elseif/else" condition using `if/2` macros, thus I'm not sure that is possible.
Well, we can achieve it by placing one more `if/2` inside `else` block, but that would probably do not look nice.

Another way to do that is to use `cond/1` macro.

### The `cond`

Once we have no `if/elseif/else` (well I think so), we can use `cond/1` instead. Let's solve the FizzBuzz problem using `cond`.

```elixir
defmodule FizzBuzz do
  def calculate(n) do
    cond do
      rem(n, 15) == 0 -> # if
        "FizzBuzz"
      rem(n, 3) == 0 -> # else if
        "Fizz"
      rem(n, 5) == 0 -> # else if
        "Buzz"
      true -> # else
        n
    end
  end
end
```

```elixir
iex> c "fizz_buzz_cond.ex"
[FizzBuzz]

iex> Enum.map([1, 3, 5, 15, 16], &FizzBuzz.calculate/1)
[1, "Fizz", "Buzz", "FizzBuzz", 16]
```
I've put some comments just to reflect the comparison with `if/elseif/else` from other programming languages.

You can also compare the implementation of FizzBuzz with [the previous implementation](http://whatdidilearn.info/2017/10/06/functions-in-elixir.html#fizzbuzz-example) of the problem without any conditions.

As we can see here, the syntax of `cond/1` is pretty simple. We are passing a block which contains `condition -> result` expressions. We can have them as much as we need.

Small side note `cond/1` will raise an `CondClauseError` exception if none of the conditions are satisfied.

```elixir
iex> cond do
...>   1 == 2 -> "Nope"
...> end
** (CondClauseError) no cond clause evaluated to a true value
```

That is why we had `true -> n` as a final `else` in our FizzBuzz solution.

### And the `case`

The syntax of the `case/2` is very similar to the switch statements from any other programming languages. 
We have an expression and we match it against other clauses we have:

```elixir
iex> case 2 + 2 do
...>   5 -> "Nope"
...>   4 -> "Yep"
...>   22 -> "What?"
...> end
"Yep"
```

As well as `cond/1` `case/2` will raise an `CaseClauseError` exception if there are no matches:

```elixir
iex(1)> case 3 + 3 do
...(1)>  4 -> "Nope"
...(1)> end
** (CaseClauseError) no case clause matching: 1
```

That is why providing a default clause would be a wise idea.

```elixir
iex> case 3 + 3 do
...>   4 -> "Nope"
...>   _ -> "I don't care"
...> end
"I don't care"
```

## Loops

Let's look at another control flow statement as loops.<br />
There are no loops in Elixir.<br />
That's it.<br />
Done.

Ok. You probably expect some explanation here.

Due to [immutability of its values](http://whatdidilearn.info/2017/09/27/a-quick-glance-on-basic-types-in-elixir.html#immutability), Elixir does not have loops in the usual for imperative languages form.

In Elixir (I think as in other declarative languages) the one way would achieve it using recursion.

```elixir
defmodule Loop do
  def puts_times(0, _message) do
    # Recursion terminator
  end

  def puts_times(n, message) do
    IO.puts message
    puts_times(n - 1, message)
  end
end
```

```elixir
iex> c "loop.ex"
[Loop]

iex> Loop.puts_times(5, "Hello Loop!")
Hello Loop!
Hello Loop!
Hello Loop!
Hello Loop!
Hello Loop!
nil
```

## Comprehensions

You may be familiar with `for-each` loop from some other languages.
Which takes an array and goes through every element of that array and does something with the element.

For Ruby it would be `for` loop:

```ruby
irb> for i in 1..3 do
irb*   puts i
irb> end
1
2
3
=> 1..3
```

Elixir provides us similar functionality (probably improved one), but here it is called "Comprehensions".
So not a loop again.

The similar example in Elixir looks like:

```elixir
iex> for i <- 1..3, do: IO.puts(i)
1
2
3
[:ok, :ok, :ok]
```
Where the `for/1` accepts "generator" to iterate through the values. 

Same as for the `if` condition we have a syntactic sugar version.

```elixir
iex> for i <- 1..3 do
...> IO.puts(i)
...> end
1
2
3
[:ok, :ok, :ok]
```

The first difference we can see comparing to Ruby it is that comprehension returns a list containing the result of the operation instead of the initial collection.
In our case it's `1..3` vs `[:ok, :ok, :ok]` (`:ok` is a result of `IO.puts`).

Let's suppose we have a collection of numbers but we want to work only with numbers less than 4.
The naive implementation might look like:

```elixir
iex> for i <- 1..5 do
...>   if i < 4, do: i
...> end
[1, 2, 3, nil, nil]
```

As we can see that is not only naive but also does not work as we expected.

For these cases comprehensions also accepts filters.

```elixir
iex> for i <- 1..5, i < 4, do: i
[1, 2, 3]
```

Now we got only permitted values as a result.

Next, let's try to nest one comprehension into another.

```elixir
iex> for i <- [1, 2] do
...>   for j <- [11, 12] do
...>     {i, j}
...>   end
...> end
[[{1, 11}, {1, 12}], [{2, 11}, {2, 12}]]
```

In this case, we are going through every combination of `i` and `j` and return is as a tuple.
In case we want to avoid nested lists we would need to pass the result to `List.flatten/1` function:

```elixir
iex> List.flatten [[{1, 11}, {1, 12}], [{2, 11}, {2, 12}]]
[{1, 11}, {1, 12}, {2, 11}, {2, 12}]
```

It works, but I would say the implemtation does not look in Elixir way. Now let's check how it can be done using single comprehension:

```elixir
iex> for i <- [1, 2], j <- [11, 12], do: {i, j}
[{1, 11}, {1, 12}, {2, 11}, {2, 12}]
```

We can provide generators one by one and comprehension will iterate through every value and return the result as a one-level list.

The last thing I want to mention is: The variable assignments inside the comprehension are treated as local variables.
That being said defining a variable with the same name does not affect the existing one.

```elixir
iex> i = 503
503

iex> for i <- 1..3, do: i*i
[1, 4, 9]

iex> i
503
```

## Summary

Let's take a quick look what did we learn here.
Now we know how to control the flow of our programs even though Elixir provides us different alternatives.
We figured out how to use `if` condition and how to implement `if/elseif/else` conditions in Elixir using `cond` macro.
We have learned that Elixir does not have loops in the traditional understanding of imperative languages and we need to use either recursions or comprehensions to iterate through collections.

We have covered almost all the basics of Elixir, but we will not stop here. We will go deeper into the world of Elixir in the following articles.
