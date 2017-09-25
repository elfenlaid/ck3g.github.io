---
layout: post
title: Postgres Sorting with NULLs
desc: "Postgres Sorting with NULLs"
keywords: "Postgres, Postgresql, NULL values, sorting"
tags: [Postgres]
---

First goes the story.

I've started learning German language and I want to learn more effectively.
One of the methods for learning words is using "Word Cards".
Another technique for learning stuff called "Spaced Repeition".
The idea behind that technique is to repeat the information right before the brain is about to forget it.
Our brain requires to repeat recently received information several times before it end up in the long term memory.
There is [an article about "Forgetting curve"](https://uwaterloo.ca/counselling-services/curve-forgetting) describing that approach.

I've build the app for learning words based on "Words Cards" and "Spaced Repetition".
I have implemented "Practice" mode, so app give me the words I need to repeat.

I have a `words` table with all my words and I use `last_practiced_at` DateTime field for tracking last word practice.
The app uses that time to for sorting words and provide "most outdated" word first.

The query looks like:

``` sql
SELECT *
FROM words
WHERE last_practiced_at < NOW() OR last_practiced_at IS NULL
ORDER BY last_practiced_at;
LIMIT 1
```

The problem is newly created words (nonpracticed) does not have last practiced time yet and the `last_practiced_at` filled with a `NULL`s.
Thus I need to run out of already practiced words first then I will be able to practice new words.
That is not exactly I want. I want to practice new words first.

Here comes the [`NULLS FIRST`](http://www.postgresql.org/docs/8.3/static/queries-order.html) option for `ORDER BY`.

``` sql
SELECT *
FROM words
WHERE last_practiced_at < NOW() OR last_practiced_at IS NULL
ORDER BY last_practiced_at NULLS FIRST;
LIMIT 1
```

Now I'm getting new words before the outdated words.
