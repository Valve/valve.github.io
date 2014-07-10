---
layout: post
title: "from object to functional — immutability"
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
not be scientifically correct, but you can think of them simply as functions that
always accept an argument, always return the result and
the computing of the result depends solely on the argument.
In other words, a pure function cannot depend on some other data, called
_state_, existing elsewhere, to influence how the result is computed.
The only thing that dictates how the result is computed is the
function argument. Pure functions cannot change the external state
either, which is called _creating side effects_.

Sometimes programmers call the external state as 'the world' and refer
to pure functions as functions that cannot depend on 'the world' or
'read the world state', nor change the world while making its job.

Why bother at all about the purity of functions? The reason for this is
composability. When all your functions are pure, you can compose large
programs from small functions. Knowing that a function is pure is knowing
that it operates only inside itself, thus providing guarantees
that it will not change the external state, i.e. will not make side effects.

Is it possible to write a real program using only pure functions?
How can you talk to the database, write to files, charge
credit cards and do all other stuff the real programs do? Functional
applications are usually built using a pure core, where the bulk of
the logic lives and a thin, impure shell, that provides access to the
pure core from the outside world. This way you have a large part
of the code that is easy to reason about, easy to test and understand.

Example of a pure function:

{% codeblock lang:ruby %}
def sum_two_numbers(a,b)
  a + b
end
{% endcodeblock %}

You can see that this function computes the result only using its
arguments.

Example of an impure function:

{% codeblock lang:ruby %}
def sum_two_numbers(a,b)
  logger.info("calculating sum of two numbers")
  a + b
end
{% endcodeblock %}

This function writes to the file system besides computing the result.
In other words, this function changes the world by creating side
effects.

Using v2 of this function you hurt composability; you limit yourself in
the ways you can use this function further in your program.

#### Immutability and purity

Now let's look why function purity demands immutability on a concrete
example. We all know that strings in ruby are [mutable](http://stackoverflow.com/q/2608493/430254).
You can mutate the string with:

{% codeblock lang:ruby %}
s = "Hello"
# mutating with '<<'
s << ", world"
# mutating with bang methods
s.upcase!
puts s
# => "HELLO, WORLD"
{% endcodeblock %}

This code fragment modifies the string in-place, mutating it.
Now let's use the string as a function argument:

{% codeblock lang:ruby %}
def upcase_string(input)
  input.upcase!
  input
end
{% endcodeblock %}

This method mutates the argument and returns it.
On the surface, this looks OK, but we have just inadvertently created a
side effect. Any external code that relied on this string will
possibly break.

Let's create an example of this:
{% codeblock lang:ruby %}

def upcase_string(input)
  input.upcase!
  input
end
current_user_name = get_current_user.name
upcased_user_name = upcase_string(current_user_name)
# ...
# ...
# somewhere else still thinking that current_user_name is downcased
if current_user_name == 'admin' 
  # this will never be true
  # ...
end
{% endcodeblock %}

You see now that in order to keep function pure we should never mutate
its arguments, but create new objects and return them instead.
Same function, but this time pure:

{% codeblock lang:ruby %}
def upcase_string(input)
  input.upcase
end
{% endcodeblock %}

Just a minor modification gives us many benefits:
we're no longer modifying the world and only returning a new string with
the required modifications.

How can we guarantee that functions do not mutate their arguments?
By making the arguments immutable, of course!

The key thing to take away here is that by making each object immutable,
we can guarantee that functions will not create side effects and will
always be pure.

Hopefully, by now I have convinced you that immutable objects are useful.
Now you probably understand that by limiting the 'reach' of the function
to only the local function scope you automatically decrease the number
of potential bugs and unpleasant surprises.
However, you may still be unsure about the performance of the immutable objects,
and think that it is wasteful to create a copy of an object each time
it needs to be modified.
The following part of the article will hopefully make everything clear.

### Immutability and primitives

Let's define what primitives are. For our purposes, we can refer to
primitives as data types, that serve as basic building blocks of the language.
Usually the primitives are directly supported by the language.
Ints, floats, characters and booleans are primitives and are usually
treated in a special way by languages.

You don't need to do something like:
{% codeblock lang:ruby%}
# in fact you can't do this in Ruby
num = Integer.new(99)
{% endcodeblock %}

You can use primitives directly:

{% codeblock lang:ruby%}
num = 99
fnum = 3.14
{% endcodeblock %}

Why does a language usually divide objects into, well, objects and
primitives? The reason is performance.
Primitives are closer to computer hardware and creating an object each time a number
is needed is slow.

However, Ruby does not have true [primitives](http://en.wikipedia.org/wiki/Primitive_data_type),
because [in Ruby, everything is an object](https://www.ruby-lang.org/en/about/).
You can call methods and properties on numbers and extend them with
user-defined methods.

### Strings

### Immutable data structures

#### Immutable collections

### Immutability and multithreading

### Immutability and copy on write

### Conclusion

#TODO: references, multiple arguments, currying, nested functions,
#defensive copying, immutable arrays
