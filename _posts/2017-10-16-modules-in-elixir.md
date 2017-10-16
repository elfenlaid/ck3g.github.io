---
layout: post
title: "Modules in Elixir"
desc: "Learn how to define and use Modules in Elixir and how to reuse their functions, macros and of course module attributes."
keywords: "Modules, import, alias, Module attributes, Elixir"
tags: [Elixir]
---

Last time we were talking about [functions](2017/10/06/functions-in-elixir.html) I have mentioned that named functions are living inside modules.
Modules provide the namespace for functions and other things defined inside.
Usually grouped by same meaning.

For example, Elixir has set of functions to work with enumerables defined in the [Enum](https://hexdocs.pm/elixir/Enum.html) module.
And [IO](https://hexdocs.pm/elixir/IO.html) module contains functions for handling Input/Output operations.

We define modules using `defmodule` keyword followed by its name and a bloc, which contains a body of the module.

```elixir
defmodule FizzBuzz do
  def print do
    IO.puts "FizzBuzz"
  end
end
```

To call a function inside module we use `ModuleName.function_name` format.

```elixir
iex> FizzBuzz.print
FizzBuzz
```

We can also define a module inside another module.

```elixir
defmodule FizzBuzz do
  defmodule MyIO do
    def print(content) do
      IO.puts content
    end
  end

  def print do
    MyIO.print "FizzBuzz"
  end
end
```

and then call the function using the dot as a separator between module names.

```elixir
iex> FizzBuzz.MyIO.print "Hello module"
Hello module
```

That means we can use the same format while we are defining nested module


```elixir
defmodule FizzBuzz.MyIO do
  def print(content) do
    IO.puts content
  end
end
```

It is worth to mention that the module name is an atom behind the scenes.
For example, if we decide to represent our module name as a string,
Elixir would prefix it with "Elixir."

```elixir
iex> to_string FizzBuzz.MyIO
"Elixir.FizzBuzz.MyIO"
```

That means we can use it to call the module in the following way:

```elixir
iex> :"Elixir.FizzBuzz.MyIO".print "Module name is an atom"
Module name is an atom
```

## import

To eliminate duplication of existing functions we can use `import` directive to bring functions and macros of one module into our current one. That means we can reuse already defined functions within the current scope.

```elixir
defmodule FizzBuzz do
  def fizz do
    "Fizz"
  end

  def buzz do
    "buzz"
  end
end

defmodule FizzBuzzUser do
  import FizzBuzz

  def fizz_buzz do
    "#{fizz}#{buzz}"
  end
end
```

We can see the result by calling our function

```elixir
iex> FizzBuzzUser.fizz_buzz
"Fizzbuzz"
```

These functions will not be available to call outside  of the module:

```elixir
iex> FizzBuzzUser.fizz
** (UndefinedFunctionError) function FizzBuzzUser.fizz/0 is undefined or private. Did you mean one of:

      * fizz_buzz/0

    FizzBuzzUser.fizz()
```

It is allowed to import only (or except) some functions. Using following format:

```elixir
import ModuleName, only: [function_name: arity, function_name: arity]
import ModuleName, except: [function_name: arity, function_name: arity]
```

We can also import only functions or only macros:

```elixir
import ModuleName, only: :functions
import ModuleName, only: :macros
```

But that works only with `only` option, `except` is not allowed in that case.

Using `only` or `except` is recommended in order to avoid importing all the functions from that module.

## alias

We can use the `alias` directive to have an alias for a module, usually to minimize typing.

Can be used in the following format: 

```elixir
alias Very.Long.ModuleName, as: ShortName
```

The option `:as` can be omitted. In that case, the aliased name would be the same as the last part of the whole module name.
From the example above it would be just `ModuleName`.
So the following expressions are equivalent:

```elixir
alias Very.Long.ModuleName, as: ModuleName
alias Very.Long.ModuleName
```

If you want to add aliases for the several modules from the same namespace, you can use the following format:

```elixir
defmodule Very.Long.ModuleName1 do end
defmodule Very.Long.ModuleName2 do end

alias Very.Long.{ModuleName1, ModuleName2}
```

Let's summarize it with an example


```elixir
defmodule VeryLongMethodNameWhichIsAnoyingToType do
  def name do
    "Some name here"
  end
end

defmodule AnotherModule do
  alias VeryLongMethodNameWhichIsAnoyingToType, as: LongName

  def name do
    LongName.name
  end
end
```

```elixir
iex> AnotherModule.name
"Some name here"
```

An alias can be used inside function body in case you don't want to expose it for the whole module.


There are two more directives `require` and `use`. They are more intended to work with macros. I will cover them when I talk about macros.

## Module attributes

We can define attributes in the module to serve as annotations and/or constants.
They work only on the top level of the module and cannot be set inside a function.
To define an attribute we are using attribute name prefixed by `@`.

```elixir
defmodule MyApi do
  @version 1

  def api_version do
    @version
  end
end
```

We can define the same attribute multiple times, but that will affect on functions defined below it.

```elixir
defmodule MyApi do
  @version 1
  def first_version, do: @version

  @version 2
  def second_version, do: @version
end

MyApi.first_version # => 1
MyApi.second_version # => 2
```

## Summary

Here we covered an introduction to the modules.
Now we know how to define them, how to call functions inside the modules.
We have also learned how to use directives for code reuse and module attributes.
Regarding attributes, there is some interesting usage which I will cover later.
