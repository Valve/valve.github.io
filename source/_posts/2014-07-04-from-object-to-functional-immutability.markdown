---
layout: post
title: "immutability in Ruby"
date: 2014-07-04 15:30
comments: true
categories: [language, ruby, immutability]
---

Like many other developers, I've been intrigued with functional
programming for a long while. I remember myself reading articles
promising programming heaven for those who are brave enough to go
functional. I bought a used [Real World Haskell](http://book.realworldhaskell.org/)
on Ebay, but sadly never finished it.
I then bought [Scala for the Impatient](http://www.amazon.com/Scala-Impatient-Cay-S-Horstmann/dp/0321774094),
but this time had the persistence to finish the book.

All these years functional programming seemed like a holy grail,
but as a true holy grail, I was afraid it was meant to stay
undiscovered.

All these years I paid my bills writing Ruby-on-Rails and JavaScript
code and never made the functional leap. I never became a full-time
Haskell or Scala developer and probably will never become one.

But you know what? It's possible to be slightly more functional with
_normal_ languages we're using every day. This article will try to demonstrate
several concrete examples where functional programming is useful
or elegant. I will show you the old way of doing things in Ruby and the
new, more functional way of doing similar things in Ruby again.

<!--more-->

Let me start by saying that this article assumes you're interested in
functional programming. It also assumes that you've probably seen
other examples of functional code before.

I'm going to split this article into several parts and each part will
elaborate upon a specific example.

## Part 1: Immutability

What is immutability? When people speak about immutability they usually
mean [immutable objects](http://en.wikipedia.org/wiki/Immutable_object).
Quoting from wikipedia:

>> an immutable object is an object whose state cannot be modified after it is created.
This is in contrast to a mutable object, which can be modified after it is created.

A very simple concept with far reaching consequences.

First let's define what 'whose state cannot be modified' really means.
At first you may think that such an object is useless. How can we possibly
use an object if we cannot change it?
Usually an immutable object creates a copy of
itself with desired modifications. The original object remains
unchanged. You will see the examples of it further in the article.

### Immutability and functional programming

Now, another foundational question: why does functional programming favor
immutable values and data structures over mutable ones? Is _real_ functional
programming possible with mutable values?
You probably know that functional programming is more than 'programming
with functions'. It also requires the functions to be _pure_.
I'm not a mathematician and  my explanation of pure functions may
not be scientifically correct, but you can think of them simply as functions that:
always accept an argument, always return a result and
the computing of the result depends solely on the input.
In other words, a pure function cannot depend on some other data,
existing elsewhere, called _state_, to influence how the result is computed.
The only thing that dictates how the result is computed is the
function's argument. Pure functions cannot change the external state
either. This is called _creating side effects_.

Sometimes programmers call the external state "the world" and refer
to pure functions as functions that cannot depend on "the world" and
read "the world" state, nor change "the world" while making its job.

Why worry at all about the purity of functions? Composability.
When your functions are pure, you can compose large
programs from small functions. Knowing that a function is pure
provides guarantees that it will not change the external state.

Is it possible to write a real program using only pure functions?
How can you talk to the database, write to files, charge
credit cards and do everything else real programs do? Functional
applications are usually built using a pure core (where the bulk of
the logic lives) and a thin, impure shell (that provides access to the
pure core from the outside world). This way you have a large part
of the code that is easy to reason about, easy to test and easy to understand.

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

This function writes to the file system in addition to computing the result.
In other words, this function changes "the world" by creating side
effects.

Using v2 of this function you hurt composability; you limit yourself in
the ways you can use this function in other parts of your program.

#### Immutability and purity

Now let's look why function purity demands immutability with a concrete
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
side effect. Any external code that relies on this string may break.

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
Same function, but this time implemented as pure:

{% codeblock lang:ruby %}
def upcase_string(input)
  input.upcase
end
{% endcodeblock %}

Just a minor modification gives us many benefits:
we're no longer modifying "the world" and only return a new string with
the required modifications.

How can we guarantee that functions never mutate their arguments?
By making the arguments immutable, of course!

The key thing to take away here is that by making each object immutable,
we can guarantee that functions do not create side effects and remain pure.

Hopefully, by now I have convinced you that immutable objects are useful.
Now you probably understand that by limiting the "reach" of the function
to only the local function's scope you automatically decrease the number
of potential bugs and unpleasant surprises.
However, you may still be unsure about the performance of immutable objects,
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
Primitives are closer to computer hardware and creating an object for
every number is slow.

However, Ruby does not have true [primitives](http://en.wikipedia.org/wiki/Primitive_data_type),
because in Ruby, [everything is an object](https://www.ruby-lang.org/en/about/).
You can call methods and properties on numbers and extend them with
user-defined methods. I will still call them primitives, because it's
what they are on a conceptual level.

On one hand, primitives behave like immutable objects
in Ruby:

{% codeblock lang:ruby%}
i = 99
puts i.object_id
# => 7
i += 1
puts i.object_id
# => 12
{% endcodeblock %}

This snippet demonstrates that you cannot modify a number. In real life
this doesn't make sense either, if you have the number 4 it's the number 4 --
eternal and beautiful. If you add 1 to it, you get completely different
number 5, the old 4 stays the same.

On the other hand, you can define your own methods and properties:

{% codeblock lang:ruby%}
class TrueClass
  attr_accessor :name
end
true.name = "one"
false.name = "two"
{% endcodeblock %}

Integers and floats are _frozen_ by default, while booleans are not.

{% codeblock lang:ruby%}
1.frozen? # true
3.14.frozen? # true
true.frozen? # false
{% endcodeblock %}

So while some primitives are not frozen, Ruby does not provide mutation
methods for them and they _usually_ can be treated as immutable objects.
You should remember that this is easily overridden (as is everything in Ruby)
and can cause potential problems.

### Strings

Before diving into the specifics of Ruby strings, let's
talk about string mutability in general. In most languages strings are
immutable: string concatenation or upcasing produces a new
string rather than modifying it in-place.

Why do language designers usually make their string implementations
immutable? To answer that we need to remember that strings are one of
the most used data structures in any programming language.

Let's consider the cases when string immutability is useful.

#### Concurrency.

This is a complex topic and I will talk about it later in
the article. What you should know at this point is that when any data
structure is immutable, it can be freely shared across threads
without any locking or synchronization. Immutable data structures
don't need synchronisation at all when used in multithreaded
environments.

Modern programming languages are designed from the ground up to be
concurrent (go, rust), and having a single string instance to be
shared across multiple threads helps to save a lot of memory and avoid
the necessity of defensive copying when passing immutable strings
around.

#### Hash table keys

More often than other data types, strings are used as keys in hash
tables. This usage demands for strings to return the same hash code
after the key and value were added to the hash table. With mutable
strings a hash table would need to copy the string in order to guarantee
the hash code staying the same. With immutable strings this is not needed.

#### Security

As I've mentioned, strings are used very frequently in any program.
This entails a special treatment in terms of security.
Strings are used when comparing user-names and passwords, storing
credit card numbers and much more. Immutable strings guarantee
that a malicious party is unable to tamper with the string after
creation.

However, there is a performance downside of immutable strings.
Mutable strings allow fast indexing and modifying  in-place,
as with regular arrays.

#### String immutability and Ruby

As with _primitives_, Ruby has no real immutable strings. To be precise,
Ruby strings are mutable behind an immutable facade. That is, most
operations on strings return new strings, while some of them still allow
in-place modification.

Consider these examples:

{% codeblock lang:ruby %}
# immutable operations
s = "hello"
puts s.object_id # 70093095097920
s += ", world"
puts s.object_id # 70093096228400
s = s.upcase
puts s.object_id # 70093096177000

# mutable operations
s = "hello"
puts s.object_id # 70093096113460
s << ", world"
puts s.object_id # 70093096113460
s = s.upcase!
puts s.object_id # 70093096113460
{% endcodeblock %}

As you can see mutable operations do not create new strings but rather
modify existing strings in-place.

##### Strings as hash keys

Earlier I mentioned that mutable strings do not make good hash keys.
Let me prove this:

{% codeblock lang:ruby %}
bad_key = "hal9000"
h = {bad_key => "Odyssey"}
h[bad_key] # "Odyssey"
bad_key << "!"
h[bad_key] # nil
{% endcodeblock %}

After I modified the string key we can no longer find the value,
because the key's hashcode has changed! Since it's so easy to mutate the
Ruby string, you can end up with a useless hash.
This is why it is not recommended to use mutable strings as hash keys.

How can we remedy it?
The first option is to freeze the string:

{% codeblock lang:ruby %}
better_key = "hall9000".freeze
h = {better_key => "Odyssey"}
better_key << "!"
# RuntimeError: can't modify frozen String
{% endcodeblock %}

A second and better option is to use [symbols](http://www.ruby-doc.org/core-2.1.2/Symbol.html),
which are immutable versions of strings often used as identifiers.

{% codeblock lang:ruby %}
best_key = :hal9000
h = {best_key => "Odyssey"}
best_key << :a
NoMethodError: undefined method `<<' for :hal9000:Symbol
{% endcodeblock %}

When using literal symbols as hash keys, Ruby provides a shorter syntax:

{% codeblock lang:ruby %}
h = {hal9000: "Odyssey"}
# hal9000: gets converted to :hal9000 =>
{% endcodeblock %}

You might say at this point, "Why can't I just use symbols instead of
strings if they're immutable equivalents?". The short answer is you may not be
able to, depending on your use case.
One reason is that symbols don't have immutable equivalents of
string's many methods, so it's inconvenient to use symbols as an immutable replacement.
Just compare the number of methods in [Symbol](http://www.ruby-doc.org/core-2.1.2/Symbol.html)
and [String](http://www.ruby-doc.org/core-2.1.2/String.html) to see the
difference.

### Immutable data structures

So far my discussion was around built-in data types and their
relationships with immutability. Real-life applications, however,
require using data structures in order to be efficient.

What is a data structure?

It's a complex question, but you can think of
it as a way to organize other, simpler data structures in a convenient or efficient
way. Some data structures are designed for ease of use, while others are
built solely with efficiency in mind.

We all know about lists, queues, hash tables, arrays, trees and many, many
more. These data structres can have both mutable and immutable implementations.

Mutable implementations are considered 'classic', because they are more
widely used, have been around for longer and generally are easier to implement.
Immutable counterparts offer advantages in concurrency and security.

While some people use 'immutable' and 'persistent' interchangeably,
they are not the same. [Persistent data structure](http://en.wikipedia.org/wiki/Persistent_data_structure)
is immutable and keeps and reuses large
parts of itself while constructing an immutable copy. As an example, you can think of a persistent
linked list that reuses its tail when appending a new node.
If you're interested in functional, persistent data structures, have a
look at [Purely functional data structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504)
by Chris Okasaki.

Let me also add that many modern programming languages that focus on
concurrency have their data structures implemented in an immutable
fashion: [Scala](http://www.scala-lang.org/api/2.11.1/#scala.collection.immutable.package)
offers both immutable and mutable collections. [Clojure](http://clojure.org/functional_programming#Functional Programming--Immutable Data Structures) and  [C#](http://msdn.microsoft.com/en-us/library/dn385366\(v=vs.110\).aspx) offer immutable collections as well.

Let's go ahead and implement a classic, mutable stack in Ruby and
then reimplement it as immutable.
A [stack](http://en.wikipedia.org/wiki/Stack_(abstract_data_type) is a data structure that follows this interface:

```
self push(item)
self pop()
item peek()
bool empty?
```

Here is mutable implementation that uses a Ruby array as a backing store:

{% codeblock lang:ruby %}
class MutableStack
  def initialize
    @store = []
  end

  def push(item)
    @store.push(item)
    self
  end

  def pop
    @store.pop
    @store
  end

  def peek
    @store[-1]
  end

  def empty?
    @store.empty?
  end
end
{% endcodeblock %}

This implementation is basically a thin wrapper around array.
Whenever you call `stack.push(item)`, you're modifying this array
in-place. This implementation possesses all the weaknesses that we discussed
previously.

Now to an immutable implementation:

{% codeblock lang:ruby %}

class ImmutableStack
  class EmptyStack
    def empty?
      true
    end

    def push(item)
      ImmutableStack.new(item, self)
    end

    def pop
      raise 'Cannot pop empty stack'
    end

    def peek
      raise 'Cannot peek empty stack'
    end
  end

  def self.empty
    EmptyStack.new
  end

  def initialize(head, tail)
    @head = head
    @tail = tail
  end

  def peek
    head
  end

  def push(item)
    ImmutableStack.new(item, self)
  end

  def pop
    tail
  end

  def empty?
    false
  end
end
{% endcodeblock %}

Usage pattern:

{% codeblock lang:ruby %}

s = ImmutableStack.empty
s = s.push(99)
s = s.push(100)
puts s.peek # 100
s = s.pop
puts s.peek # 99
s.peek # Cannot peek empty stack (RuntimeError)

{% endcodeblock %}

Each destructive operation does not mutate the stack but
rather returns a copy of itself with the required modifications.
What's more, it reuses a large portion of itself while doing so, thus
making this stack a persistent data structure.

Unfortunately, Ruby does not allow you to directly create private
constructors and users can potentially call

{% codeblock lang:ruby %}

ImmutableStack.new(1, ImmutableStack.empty)

{% endcodeblock %}

If you want a good ruby library of immutable collections, I suggest
using [hamster](https://github.com/hamstergem/hamster).

#### Immutable data structures and multithreading

When writing a multi-threaded applications, follow these rules:

1. Avoid sharing data across threads.
2. If you have to share your data across threads, make this data immutable.
3. If you can't avoid sharing mutable data, synchronize access to that
   data with synchronization constructs, such as [Mutex](http://www.ruby-doc.org/core-2.1.1/Mutex.html).

In our two stack implementations it is safe to share an
immutable version across multiple threads, because they will not be able to
modify it in place. Whenever a thread makes a `push` or a `pop`, a
new instance of the stack is created and returned so that the existing
instance is never changed.

### Conclusion

Now that you've read the article, you might have the impression that
immutability is a silver bullet.  It is not.
It is only one possible way to design software and has its own strengths
and weaknesses. Immutability let's you design your functions and data
structures in a new way, gaining much and losing much too.
We've all been living in a world where the sequential computation was
the de-facto standard.
In the past immutability was not worth it.
For a single core computer, immutability has too much overhead.
You must carefully control the state and
pay close attention to reusing and copying in order to be efficient.
The performance impact that some of the immutable data structures incur can
be too significant.

But the world is changing and the sequential model is disappearing.
We all have smartphones with 2 or 4 cores. Our smart watches will have 8
cores in a couple of years, which means that concurrent will become
the new sequential. If we want to exploit the power of modern hardware, we need
to embrace the concurrent way of doing things. This is where
immutability advantages outweigh the bad parts.
I think that immutability is the way you should design your
software now in order to be prepared for the concurrent future.
