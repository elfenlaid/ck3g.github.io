---
layout: post
title: The "jj" method
desc: "The jj method of ruby"
keywords: "Ruby, json, jj, pretty_generate"
tags: [Ruby, JSON]
---

Just as you can use `pp` method to "pretty print" passed object. You can use `jj` to pretty print passed objects as a JSON.

The usage of it is pretty simple. All you need to pass object (or set of objects):

``` ruby
require 'json'

jj(
  name: "Best Lunches in Town",
  working_hours: { opens: "11:00 am", closes: "20:00 pm" },
  menu: {
    main_course: "Schnitzel with Potato",
    appetizer: "Chicken Broth",
    desser: "Pana Cota"
  }
)
```

And the output would a well formatted JSON:

``` json
{
  "menu": {
    "main_course": "Schnitzel with Potato",
    "appetizer": "Chicken Broth",
    "desser": "Pana Cota"
  },
  "working_hours": {
    "opens": "11:00 am",
    "closes": "20:00 pm"
  },
  "name": "Best Lunches in Town"
}
```

The `jj` method uses [`JSON.pretty_generate`](http://ruby-doc.org/stdlib-2.3.0/libdoc/json/rdoc/JSON.html#method-i-pretty_generate) behind the scene.
It simply iterates through the passed arguments and calls `pretty_generate` for each of them.

That's prretty much it.
