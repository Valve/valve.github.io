---
layout: post
title: "from object to functional by example"
date: 2014-07-04 15:30
comments: true
categories: [functional, object, programming, language, ruby, rust,
scala]
---

Like many other developers, I've been intrigued with functional
programming for a long while. I remember myself reading articles
promising programming heaven for those who are brave enough to go
functional. I bought a used [Real World Haskell](http://book.realworldhaskell.org/)
on Ebay, but sadly never finished it.
I then bought [Scala for the Impatient](http://www.amazon.com/Scala-Impatient-Cay-S-Horstmann/dp/0321774094),
but this time had the persistence to finish the book.

All these years functional programming seemed like a holy grail for me,
but as a true holy grail, I was afraid it was meant to stay
undiscovered to me.

All these years I paid my bills writing Ruby-on-Rails and JavaScript
code and never made this functional leap. I never became a full-time
Haskell or Scala developer and probably will never become one.

But you know what? It's possible to be slightly more functional with
_usual_ languages we're using every day. This article will try to demonstrate
to you several concrete examples where functional programming is useful
or elegant. I will show you the old way of doing things in Ruby and the
new, more functional way of doing same thing in Ruby again.

For the sake of completeness, I'll be using Scala and Rust in these
examples as well.  Scala is a known functional language, but why Rust?

Just because I like it.

<!--more-->

Let me start by saying that this article assumes you're interested in
functional programming. It also assumes that you've probably seen
other examples of functional code before.

I'm going to split this article into several parts and each part will
explain a specific example.

## Part 1: Immutability

What is immutability? When people speak about immutability they usually
mean [immutable objects](http://en.wikipedia.org/wiki/Immutable_object).
Quoting from wikipedia:

>> an immutable object is an object whose state cannot be modified after it is created.
This is in contrast to a mutable object, which can be modified after it is created.

Very simple concept with far going consequences.

First let's define what 'whose state cannot be modified' really means.
At first you may think that such object is useless, how can we possibly
use an object if we cannot change it?
Usually what happens is that an immutable object creates a copy of
itself with desired modifications. The original object always stays
unchanged. You will see the examples of it further in the article.

### Immutability and functional programming

Now another foundational question: why does functional programming favor
immutable values and data structures over mutable? Is _real_ functional
programming possible with mutable values?
You probably know that functional programming is more than 'programming
with functions'. It also requires the functions to be _pure_.
I'm not a mathematician and probably my explanation of pure functions will
not be scientifically correct. But you can think of them simply as functions that
always accept an argument, always return the result and
the computing of the result depends solely on the argument.
In other words, a pure function cannot depend on some other data, called
_state_, existing elsewhere, to influence how the result is computed.
The only thing that dictates how the result is computed is the
function argument. Pure functions cannot change the external state
either.

Sometimes programmers call the external state as 'the world' and refer
to pure functions as functions that cannot depend on 'the world' or
'read the world state', nor change the world while making its job.

Example if an impure function:

{% codeblock lang:ruby %}
def calculate_ruble_rate()
end
{% endcodeblock %}


### Immutable primitive objects

### Strings

### Immutable data structures

#### Immutable collections

### Immutability and multithreading

### Immutability and copy on write

### Conclusion
