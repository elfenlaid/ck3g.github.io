---
layout: post
title: The "jj" method
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

That's prretty much it.
