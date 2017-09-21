---
layout: post
title: "Pattern matching in Elixir"
desc: "We are discovering a beatiful world of Pattern Matching"
keywords: "Pattern Matching, assignment, assignment statement, Elixir, Phoenix, learning Elixir"
---

What is one of the most common things in the programming languages? Of course, it is the assignment statement.
Without an ability to assign values to variables and constants most of the programming languages would have hard times.

Let's talk about that in Elixir.

<br />

Every developer familiar with assignment statement. Here how it is working in Ruby language:

```ruby
→ irb
> a = 1 # We are assigning value `1` to variable `a`
=> 1
> 1 = a # We are trying to assign value of variable `a` to `1`
SyntaxError: (irb):2: syntax error, unexpected '=',
expecting end-of-input
1 = a
   ^
```

Of course assigning a variable to a value does not make a lot of sense. Who on earth would to that.

Let's check the same behavior in the Interactive Elixir `iex`:

```elixir
→ iex
> a = 1
1
> 1 = a
1
```

Ehm?

```elixir
> 2 = a
** (MatchError) no match of right hand side value: 1
```

"No match of right hand side". What the hell?<br />
Ok. Calm down. Let me explain that.

The point is that there is no assignment in Elixir.

"Hey, aren't you just said it does not make sense for programming languages to have no assignment functionality?"

Yes. But Elixir does not have it. The equal sign in Elixir is the "Pattern matching". It tries to match left-hand side and a right-hand side. Don't worry, you will get it soon.

If elixir variable is on the left-hand side of the equal sign, then the value of right-hand side can be "assigned" to this variable. That also means those variables can be "re-assigned" once we are trying to match them with a different value.

What is why the following example is working in Elixir:

```elixir
> a = 1
1
> a = 2
2
```

But we can also force pattern matching if needed by using `^` Pin Operator before the variable name.

```elixir
> a = 1
1
> ^a = 2
** (MatchError) no match of right hand side value: 2
```

<br />

Let's look at more complex examples. For example, we have a list of fruits.

```elixir
> fruits = ["Apples", "Oranges", "Pears"]
["Apples", "Oranges", "Pears"]
```

We are matching it with undefined variable `fruits` and now it contains the list of the fruits.

Let's suppose we are trying to extract them into separate variables. We can achieve that using pattern matching.

```elixir
> [apples, oranges, pears] = fruits
["Apples", "Oranges", "Pears"]
> apples
"Apples"
> oranges
"Oranges"
> pears
"Pears"
```

As soon as the structure on the left-hand side can be matched with the content of the `fruits` list on the right-hand side, we can assign every value to our new variables. That means first value goes to `apples`, second to `oranges` and so on.

<br />

Let's suppose we do not care about "Pears" right now. So we can write:

```elixir
> [apples, oranges, "Pears"] = fruits
["Apples", "Oranges", "Pears"]
> apples
"Apples"
> oranges
"Oranges"
```

In this case `apples` matches with `"Apples"`, `oranges` with `"Oranges"` and `"Pears"` with `"Pears"`.

You can say: "But that is pretty boring to write `"Pears"` every time we want to ignore the value".
And you would be right. Fortunately, Elixir has an answer for that.

<br />

If you don't care about some values you can match them to special underscore `_` variable.
It is special because you cannot read from that value in Elixir.

```elixir
> [apples, oranges, _] = fruits
["Apples", "Oranges", "Pears"]
> apples
"Apples"
> oranges
"Oranges"
> _
** (CompileError) iex:8: unbound variable _
```

<br />

You can also prefix some variables with the underscore to increase readability. But once you try to access it you would get a warning:

```elixir
iex(2)> [_, _, _pears] = fruits
["Apples", "Oranges", "Pears"]
iex(3)> _pears
warning: the underscored variable "_pears" is used after being set.
A leading underscore indicates that the value of the variable should
be ignored.  If this is intended please rename the variable to remove
the underscore
  iex:3

"Pears"
```

<br />

Ok. What if our `fruits` list is huge, but we are care only about "Apples" and "Oranges"? Does that mean we need to ignore every other value in the list?

No, of course not. That would be madness.

<br />

Going a little bit forward we can split our list to "head" and "tail" and match it.

```elixir
> fruits = ["Apples", "Oranges", "Pears", "Bananas"]
["Apples", "Oranges", "Pears", "Bananas"]
> [apples, oranges | _rest] = fruits
["Apples", "Oranges", "Pears", "Bananas"]
> apples
"Apples"
> oranges
"Oranges"
```

So basically the `_rest` would contain "Pears" and "Bananas".

<br />

It's also worth to mention that it's not possible to match the same variable to the different values. Following example will not work:

```elixir
> [apples, apples | _] = fruits
** (MatchError) no match of right hand side value:
    ["Apples", "Oranges", "Pears", "Bananas"]
```

First-time `apples` is being bound to `"Apples"` value and second time we are trying to match it with `"Oranges"`.
What is why we are getting `MatchError` here.

<br />


The fun does not stop here. We can also use pattern matching in function arguments.

Let's take a look at this function:

```elixir
defmodule Fruits do
  def i_need_some_fruits(fruits) do
    [apples, oranges | _] = fruits
    IO.puts("I am not comparing #{apples} and #{oranges} here")
  end
end

> Fruits.i_need_some_fruits(["Apples", "Oranges", "Pears"])
I am not comparing Apples and Oranges here
```

<br />

The only reason why do we need `fruits` is to pattern match `apples` and `oranges` from that variable.
That means we can match them directly in the arguments:

```elixir
defmodule Fruits do
  def i_need_some_fruits([apples, oranges | _] = fruits) do
    IO.puts("I am not comparing #{apples} and #{oranges} here")
  end
end
```

<br />

As soon as we are not using `fruits` in the function body we can also throw it away.

```elixir
defmodule Fruits do
  def i_need_some_fruits([apples, oranges | _]) do
    IO.puts("I am not comparing #{apples} and #{oranges} here")
  end
end

> Fruits.i_need_some_fruits(["Apples", "Oranges", "Pears"])
I am not comparing Apples and Oranges here
```


<br />

What did we learn here? <br />
The pattern matching it is not just an assignment but a pretty powerful tool with a lot of abilities.
We also saw a bunch of examples of how to use it in different cases.
As any other tool, using it right, that can help us to write code in different (and I hope better) ways.
