---
layout: post
title: "A quick glance on basic types in Elixir"
desc: "Discover the basic types of Elixir from simple once to more complex"
keywords: "Basic Types, Integer, Atom, Boolean, String, Tuples, List, Map, Immutability"
tags: [Elixir]
---

Let's talk about basic types in Elixir.
If you are familiar with other programming languages you can see a lot of similarities in Elixir as well.

Before we dive into available types I would like to mention about one of the helpers supported by Interactive Elixir.
That helper is `i`. Using that helper we can get information about given term, or, if that term is omitted, receive information about the previous expression.

For example:

```
iex(1)> 503
1
iex(2)> i
# ---- or ----

iex(3)> i 503

# ---- Gives us ----
Term
  503
Data type
  Integer
Reference modules
  Integer
Implemented protocols
  IEx.Info, Inspect, List.Chars, String.Chars
```

From here we can see that `503` is a type of `Integer`.

Now when we know how to get information about any expression we can start with our first set of types.

## Numbers

Numbers in Elixir consist of Integers and Floats:

```
iex(1)> i 10
Term
  10
Data type
  Integer

iex(2)> i 10.5
Term
  10.5
Data type
  Float
```

Those numbers are also numbers in scientific notation:

```elixir
iex(3)> 1.0e10
1.0e10
iex(4)> 1.0e-10
1.0e-10
```

Integers, which by default are decimals, can be also represented as binary, octal and hexadecimal numbers:

```elixir
iex(1)> 0b101
5

iex(2)> 0o17
15

iex(3)> 0x1f7
503
```

For reading purposes numbers can be written using `_` underscores. For example:

```elixir
iex(1)> 1_000_000
1000000
```

I think now we have a basic understanding of numbers and we can move on.

## Ranges 

Ranges are just two integers connected with `..` two dots in between. Those numbers represent start and end of the range.

```elixir
iex(1)> 1..10
1..10
```

## Strings

A string is a sequence of characters enclosed in `"` double quotes:

```
iex(1)> "hello"
"hello"

iex(2)> i
Term
  "hello"
Data type
  BitString
Byte size
  5
Description
  This is a string: a UTF-8 encoded binary. It's printed surrounded by
  "double quotes" because all UTF-8 encoded codepoints in it are printable.
Raw representation
  <<104, 101, 108, 108, 111>>
Reference modules
  String, :binary
Implemented protocols
  IEx.Info, Collectable, Inspect, List.Chars, String.Chars
```

A multiline string can be represented with tripple double quotes:

```elixir
iex(3)> """
...(3)> hello
...(3)> world
...(3)> """
"hello\nworld\n"
```

Be careful, especially if you came from Ruby or JavaScript background. The text surrounded by `'` single quotes is not a string in Elixir.

```
iex(41)> 'hello'
'hello'
iex(42)> i
Term
  'hello'
Data type
  List
Description
  This is a list of integers that is printed as a sequence of characters
  delimited by single quotes because all the integers in it represent valid
  ASCII characters. Conventionally, such lists of integers are referred to as
  "charlists" (more precisely, a charlist is a list of Unicode codepoints,
  and ASCII is a subset of Unicode).
Raw representation
  [104, 101, 108, 108, 111]
Reference modules
  List
Implemented protocols
  IEx.Info, Collectable, Enumerable, Inspect, List.Chars, String.Chars
```

Yes. It is a "List". We will talk about lists soon.

## Atoms

Atom is a constant started with `:` followed by its name which actually represents its value.
The value/name can consists of numbers, letters, underscores and `@` sign. Can be also ended with `!` or `?` signs.
You can also turn a string into an atom by adding leading `:`.

These are all atoms:

```elixir
:atom, :me@atom, :me_atom, :"string-atom", :atom?, :atom!
```

## Booleans

Elixir provides two boolean values: `true` and `false`. Those values are actually atoms behind the scene:

```elixir
iex(1)> true == :true
true
iex(2)> false == :false
true
```

There are only two false values in Elixir. Which is `false` (`:false`) or `nil`. Any other value treated as a truthy value.

## Tuples

Tuple is a sequence of values separated by comma and surrounded by curly brackets:

```elixir
iex(1)> {:ok, "Success", 200}
{:ok, "Success", 200}
```

## Lists

Lists even though they look quite similar to arrays in other programming languages, they are a different structure.
You will see it soon.
A list is a sequence of values separated by the comma and surrounded by square brackets:

```elixir
iex(1)> [:ok, "Success", 200]
[:ok, "Success", 200]
```

Keep in mind that Elixir does not support `[]` square-brackets way to access elements of the list by its index (unless it is a keyword list, but later about that as well).

```elixir
iex(63)> [:ok, "Success", 200][0]
```
```
** (ArgumentError) the Access calls for keywords expect the key to be an atom, got: 0
    (elixir) lib/access.ex:329: Access.get/3
```

To retrieve elements from the list we can use either [Pattern Matching](/2017/09/21/pattern-matching-in-elixir.html) or `hd/1` and `tl/1` functions.

```elixir
iex(1)> [result, _, _] = [:ok, "Success", 200]
[:ok, "Success", 200]
iex(2)> result
:ok

iex(3)> head = hd([:ok, "Success", 200])
:ok

iex(4)> tail = tl([:ok, "Success", 200])
["Success", 200]
```

Remember the list of characters enclosed in `'` single quotes?
Now we can look at that from the other side:

```elixir
iex(1)> [104, 101, 108, 108, 111]
'hello'
```

but

```elixir
iex(2)> [104, 101, 108, 108, 1111]
[104, 101, 108, 108, 1111]
```

When list represent printed ASCII character, Elixir will print it as a "charlist".

## Lists vs Tuples

Lists and Tuples look quite similar. Why do they both exist in Elixir as separate types then? The answer is in internal implementation. Lists by themselves are Linked lists, that means every element holds its value and the pointer to the next element.

Tuples, on the other hand, are stored contiguously in memory. That means it is a very fast operation to get a size of the tuple or its element by index.
Getting a length of the list is a linear operation, that means Elixir need to traverse the whole list to get its size.

## Keyword lists

Lists containing key/value pairs are known as Keyword lists:

```elixir
iex(1)> [status: :ok, message: "Success"]
[status: :ok, message: "Success"]
```

Which is by the fact is list of tuples:

```elixir
iex(2)> [{:status, :ok}, {:message, "Success"}] == [status: :ok, message: "Success"]
true
```

Now, once we have the keyword list, we can access elements using `[]` square-brackets notation:

```elixir
[status: :ok, message: "Success"][:status]
:ok
```

## Maps

A map is also collection of key/value pairs and can be represented in the following way:

```elixir
iex(1)> person = %{ "first_name" => "John", "last_name" => "Doe", "age" => 35 }
%{"age" => 35, "first_name" => "John", "last_name" => "Doe"}
```

This map has keys represented as strings. To access values of these keys we can use `[]` square-bracket syntax.

```elixir
iex(2)> person["age"]
35
iex(3)> person["first_name"]
"John"
```

The keys in the map can be also represented as atoms:

```elixir
iex(4)> person = %{ full_name: "Alice Smith", age: 30 }
%{age: 30, full_name: "Alice Smith"}
```

Once we have the map with atoms as keys we can also use dot notation to access its values: 

```elixir
iex(5)> person[:age]
30
iex(6)> person.age
30
iex(7)> person.full_name
"Alice Smith"
```

Even though Maps looks similar to Keyword lists they also have a difference.
Unlike Keyword lists, Maps contain only unique keys.
That means Keyword list can have their keys to be repeated and Map cannot have repetitive keys.

## Binaries

Sometimes you may need to have a sequence of bits and bytes. You can achieve it by enclosing a list of bytes in `<< >>` separated by the comma:

```elixir
iex(1)> << 10, 2, 255 >>
<<10, 2, 255>>
```

By default it can contains values from 0 to 255:

```elixir
iex(2)> << 10, 2, 260 >>
<<10, 2, 4>>
```

But that limit can be increased by explicitly specify how much bits should be used to store that value:

```elixir
iex(3)> << 10, 2, 260::size(16) >>
<<10, 2, 1, 4>>
```

## How to check if value belongs to particular type

For each type Elixir provides fnctions in following format: `is_<type>/1`. <br />
Those functions return `true` or `false` if the value type is the same as function's type.

For example:

```elixir
iex(1)> is_integer(1)
true
iex(2)> is_integer(1.0)
false
iex(3)> is_float(1.0)
true
iex(4)> is_boolean(true)
true
iex(5)> is_atom(:atom)
true
iex(6)> is_list([1, 2, 3])
true
```

And so on.

## Immutability

It is worth to mention that all the values in Elixir are immutable,
that being said that there is absolutely no way to change the current (for example) list.
Every time you get "head" and "tail" of the list, you get a completely new copy of that list.

That gives some advantages working with variables.
Once you assign the value to a variable and pass it to any function you can be absolutely sure this value remains the same and won't be accidentally changed.


## Instead of summary

That was a quick glance on the basic types of Elixir.
We will use them more and we will talk about several of them separately with a bunch of other examples.
