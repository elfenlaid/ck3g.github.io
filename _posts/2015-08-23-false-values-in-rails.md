---
layout: post
title: False values in Rails
desc: "False values in Rails"
keywords: "Rails, Ruby, Ruby on Rails, False values, True values"
tags: [Rails]
---

As we know Ruby language has only two **false** values `false` and `nil`.
Any other value in Ruby treated as **true**.

But sometimes we need to pass true/false values into params from checkboxes or
from custom url param into our API endpoint.
Our url may looks like `/products.json?simple_view=false` then we get our param's value as String

``` ruby
> params[:simple_view]
=> "false"
```

That will prevents us to build condition like
``` ruby
if params[:simple_view]
```
in that case we will face unexpected behavior. Yes we can describe condition as
``` ruby
if params[:simple_view] != "false"
```

and that will work until someone pass `simple_view=f` or `simple_view=off`.
We can extend our condition and soon it will looks ugly.

On the other hand when we're using checkbox on our HTML page and that checkbox is binded to one of columns of the Model,
Rails treats "off" and "0" values as **false**.
Rails knows what exactly do we mean by passing values like these becase of [FALSE_VALUES](https://github.com/rails/rails/blob/55320fa9eb73498e55475d187787c135613441ab/activerecord/lib/active_record/connection_adapters/column.rb#L8) list.

``` ruby
> ActiveRecord::ConnectionAdapters::Column::FALSE_VALUES
=> #<Set: {false, 0, "0", "f", "F", "false", "FALSE", "off", "OFF"}>
```

Now we can describe our condition as:

``` ruby
unless ActiveRecord::ConnectionAdapters::Column::FALSE_VALUES.include?(params[:simple_view])
```

It's looks quite verbose so we can extract it into helper method.
