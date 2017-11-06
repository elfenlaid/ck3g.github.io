---
layout: post
title: "More on Maps and Structs in Elixir"
desc: "Learn how to use use Maps and Structures. How to update and extend them"
keywords: "Elixir, Maps, Structs"
tags: [Elixir]
---

Let's talk a little bit more about Maps. I've already covered basics in the one of my [previous articles](http://whatdidilearn.info/2017/09/27/a-quick-glance-on-basic-types-in-elixir.html#maps).
Now it is time to go deeper and discover how we can update Maps and add new items to it.

## Maps

Imagine we have a following Map:

```elixir
iex> john = %{ "first_name" => "John", "last_name" => "Doe", "age" => 35 }
%{"age" => 35, "first_name" => "John", "last_name" => "Doe"}
```

Luckily for John, he has a Birthday today, that means he grew up one year more and we need to update his age.


### Update Maps

One of the ways to update a Map is to use following format:

```elixir
new_map = %{ old_map | key => value, ... }
```

Technically we are not updating a Map because Maps in Elixir as any other types are immutable.
Thus we are getting completely new Map instead.

```elixir
iex> john = %{ john | "age" => 36 }
%{"age" => 36, "first_name" => "John", "last_name" => "Doe"}
```

The downside (or is it upside?) of this approach is that it does not insert a new item into the Map if we are providing a non-existing key. 

```elixir
iex> john = %{ john | "title" => "Mr." }
** (KeyError) key "title" not found in: %{"age" => 36, "first_name" => "John", "last_name" => "Doe"}
    (stdlib) :maps.update("title", "Mr.", %{"age" => 36, "first_name" => "John", "last_name" => "Doe"})
    (stdlib) erl_eval.erl:255: anonymous fn/2 in :erl_eval.expr/5
    (stdlib) lists.erl:1263: :lists.foldl/3
```

### Add new items

If we want to add new item to the Map we need to use [`Map.put_new/3`](https://hexdocs.pm/elixir/Map.html#put_new/3) function.

```elixir
iex> john = Map.put_new(john, "title", "Mr.")
%{"age" => 36, "first_name" => "John", "last_name" => "Doe", "title" => "Mr."}
```

Again, this function does not change our Map but produces completely new one instead.
We should always remember that when we are writing our code in Elixir.
So we need to assign the result of the function into a new variable. By the way, ["assign" is also not the right term for Elixir](http://whatdidilearn.info/2017/09/21/pattern-matching-in-elixir.html), but you've got what I mean here.

There is yet another function we can use to insert new values into the Map - [`Map.put/3`](https://hexdocs.pm/elixir/Map.html#put/3).

However, it not only inserts new values but also updates the existing ones.

```elixir
iex> john = %{"age" => 35, "first_name" => "John", "last_name" => "Doe"}
%{"age" => 35, "first_name" => "John", "last_name" => "Doe"}

iex> john = Map.put(john, "title", "Mr.")
%{"age" => 35, "first_name" => "John", "last_name" => "Doe", "title" => "Mr."}

iex> john = Map.put(john, "age", "36")
%{"age" => "36", "first_name" => "John", "last_name" => "Doe", "title" => "Mr."}
```

Depending on our needs we can either use `Map.put_new/3` or `Map.put/3` to add new items to the Map.

Now as we know how to update values and add new items it is time to learn how to remove certain keys from the Map.

### Delete keys

Elixir provides us [`Map.delete/2`](https://hexdocs.pm/elixir/Map.html#delete/2) to achieve that.

```elixir
iex> john
%{"age" => 36, "first_name" => "John", "last_name" => "Doe", "title" => "Mr."}
iex> john = Map.delete(john, "title")
%{"age" => 36, "first_name" => "John", "last_name" => "Doe"}
```

Great. Now we know how to work with the Maps. That is a good foundation for our next topic.

## Structs

Structs are some kind of wrapper around maps which provides additional functionality to the last.

Let's update our John's example using a Struct.
To define a Struct we need to define a Module, which would become the name of that structure. Then we use `defstruct` keyword followed by available keys.

```elixir
iex> defmodule Person do
...>   defstruct first_name: "", last_name: "", age: nil, adult: true
...> end
{:module, Person,
 <<70, 79, 82, 49, 0, 0, 8, 92, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 0, 234,
   0, 0, 0, 22, 13, 69, 108, 105, 120, 105, 114, 46, 80, 101, 114, 115, 111,
   110, 8, 95, 95, 105, 110, 102, 111, 95, 95, ...>>,
 %Person{adult: true, age: nil, first_name: "", last_name: ""}}

iex> john = %Person{first_name: "John", last_name: "Doe", age: 35}
%Person{adult: true, age: 35, first_name: "John", last_name: "Doe"}

iex> bob = %Person{first_name: "Bob", age: 40}
%Person{adult: true, age: 40, first_name: "Bob", last_name: ""}
```

By defining a Struct in the following way, we are specifying the fields this structure has.
That being said we are defining the particular structure of the attributes and we are not allowed to use keys which were not specified.

```elixir
iex> alice = %Person{first_name: "Alice", address: "White Hall Str. 503"}
** (KeyError) key :address not found in: %Person{adult: true, age: nil, first_name: "Alice", last_name: ""}
    (stdlib) :maps.update(:address, "White Hall Str. 503", %Person{adult: true, age: nil, first_name: "Alice", last_name: ""})
    iex: anonymous fn/2 in Person.__struct__/1
    (elixir) lib/enum.ex:1811: Enum."-reduce/3-lists^foldl/2-0-"/3
    expanding struct: Person.__struct__/1
    iex: (file)
```


### Update Structs

As soon as a Struct is a Map underneath we can update it using the same approach:

```elixir
iex> john
%Person{adult: true, age: 35, first_name: "John", last_name: "Doe"}

iex> john = %Person{ john | age: 36 }
%Person{adult: true, age: 36, first_name: "John", last_name: "Doe"}
```

Following code also works fine. Even if it might look a little bit weird:

```elixir
iex> john = Map.put(john, :age, 37)
%Person{adult: true, age: 37, first_name: "John", last_name: "Doe"}
```

However if we try to use `Map.put_new/3` or `Map.delete/2` here (which is not permitted) we would get following result:

```elixir
iex> Map.put_new(john, :title, "Mr.")
%{__struct__: Person, adult: true, age: 37, first_name: "John",
  last_name: "Doe", title: "Mr."}
  
iex> Map.delete(john, :adult)
%{__struct__: Person, age: 37, first_name: "John", last_name: "Doe"}
```

### Define functions

One of the additional advantages of the Struct over the Map is that you can define functions for the Struct.
That is probably the reason why it's wrapped into the module.

```elixir
iex> defmodule Person do
...>   defstruct first_name: "", last_name: "", age: nil
...>
...>   def has_discount?(person) do
...>     person.age != nil && person.age < 18
...>   end
...>
...>   def full_name(person) do
...>     "#{person.first_name} #{person.last_name}"
...>   end
...> end
{:module, Person,
 <<70, 79, 82, 49, 0, 0, 11, 192, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 1, 89,
   0, 0, 0, 35, 13, 69, 108, 105, 120, 105, 114, 46, 80, 101, 114, 115, 111,
   110, 8, 95, 95, 105, 110, 102, 111, 95, 95, ...>>, {:full_name, 1}}

iex> john = %Person{first_name: "John", last_name: "Doe", age: 35}
%Person{age: 35, first_name: "John", last_name: "Doe"}

iex> Person.has_discount?(john)
false

iex> Person.full_name(john)
"John Doe"
```


## Summary

Now we have several more items in our arsenal of knowledge.
What to use in your programmes Maps or Structs it's up to you.
I think once you want a well-defined type with a predefined set of attributes, go with the Structs.
If you need something short-lived and anonymous use Maps.
