---
layout: post
title: "Constant resolution in Ruby"
date: 2013-10-26 11:47
comments: true
categories: ruby
---

Ruby constant resolution has always been somewhat confusing to me.
In this article I'm going to demistify it for myself and hopefully help other readers.

<!--more--> 

## What is a constant?

Ruby constant is anything that starts with a capital.

{% codeblock lang:ruby %}

PI = 3.1415

MINUTES_IN_ONE_HOUR = 60

LOOK_MA = "I'm a constant!"

module A
end

class Person
end

module Screen::Widget::Button
end

{% endcodeblock %}

Yes, regular `ALL_CAPITAL` are constants, module and class names are constants too.

## How Ruby searches constants.

When Ruby tries to resolve a constant, it starts looking in current lexical scope by searching the current module or class. If it can't find it there, it searches the enclosing scope and so on.

It's easy to see the lexical scopes search chain with [Module::nesting](http://ruby-doc.org/core-2.0.0/Module.html#method-c-nesting) method:

{% codeblock lang:ruby %}

module A
  A_CONSTANT = 'I am defined in module A'
  module B
    module C
      def self.inspect_nesting

        puts Module.nesting.inspect
        puts A_CONSTANT
      end
    end
  end
end

A::B::C.inspect_nesting 
# => [A::B::C, A::B, A]
# => I am defined in module A

{% endcodeblock %}

`Module::nesting` returns an array of searcheable lexical scopes, starting from current.
In above case the search for `A_CONSTANT` starts from module C, then goes to enclosing scope - module B, and then to module A where it finally finds it.

## Nesting modules using alternative syntax

You've probably seen the alternative way of defining the enclosing modules:

{% codeblock lang:ruby %}

module Screen
  DEFAULT_RESOLUTION = [1024, 768]
  module Widgets
    module MacOS
    end
  end
end

# Alternative syntax

module Screen::Widgets::MacOS::Button
  def self.inspect_nesting
    puts Module.nesting.inspect
    puts DEFAULT_RESOLUTION
  end
end

Screen::Widgets::MacOS::Button.inspect_nesting
# => [Screen::Widgets::MacOS::Button]
NameError: uninitialized constant Screen::Widgets::MacOS::Button::DEFAULT_RESOLUTION
  from (irb):26:in `inspect_nesting'
  from (irb):29
{% endcodeblock %}

See the difference?  Constant resolution only uses the innermost module for searching, ignoring the enclosing scopes. By defining the modules with this shorter syntax you lose the ability to search for constants in enclosing scopes.

## Inheritance

Enclosing scopes is the first place where Ruby searches the constants. Second place is the inheritance hierarchy. Consider this code:

{% codeblock lang:ruby %}

class Person
  DRIVING_LICENSE_AGE = 18
end

class BusDriver < Person
  def can_drive_from
    DRIVING_LICENSE_AGE
  end
end

bus_driver = BusDriver.new
puts bus_driver.can_drive_from

# => 18

{% endcodeblock %}

### Mixins

Ruby can mixin modules into classes as an alternative to inheritance. When a class mixes in a module, this module inserts itself between the class being mixed in and the parent class in the inheritance hierarchy. The simple way to see this is using [ancestors](http://ruby-doc.org/core-2.0.0/Module.html#method-i-ancestors) method.

{% codeblock lang:ruby %}

module Insurable
  LIFE_INSURANCE_AMOUNT = 150_000
end

class Person
  DRIVING_LICENSE_AGE = 18
end

class BusDriver < Person
  include Insurable
  def can_drive_from
    "Can drive from #{DRIVING_LICENSE_AGE}, with life insurance of $#{LIFE_INSURANCE_AMOUNT}"
  end
end

puts BusDriver.ancestors.inspect
puts BusDriver.new.can_drive_from

# => [BusDriver, Insurable, Person, Object, Kernel, BasicObject]
# => Can drive from 18, with life insurance of $150000

{% endcodeblock %}

What's going on here? We've defined a base class `Person`, a child class `BusDriver` that inherits from `Person`. We also defined a `Insurable` module which we mixed into our `BusDriver` class. When we call the `ancestors` class method, we see the `BusDriver` class first, then `Insurable` module which was wedged between `BusDriver` and `Person`. Then goes the `Person` class, then, obviously, `Object`. This is all nice and clear.

But why do we see `Kernel` between `Object` and `BasicObject`? This is because `Kernel` is a module that is mixed into `Object` thus inserting itself into the inheritance hierarchy. This `ancestors` array is how the name resolution works throughout the inheritance chain.

## Full search path

Now that you've seen the inheritance part of the name search, you can see the full picture:

```ruby
# searching from left to right
full_path = [Module.nesting + Module.ancestors].uniq
```

## const_missing method

When Ruby has finished searching the constants up the nesting and ancestors chain and didn't find it, it gives the calling code the last chance by calling the [const_missing](http://ruby-doc.org/core-2.0.0/Module.html#method-i-const_missing) method.

{% codeblock lang:ruby %}

module Person
  def self.const_missing(name)
    puts "Oh me oh my, can't find the constant: #{name}"
  end
end

Person::LOL
# => Oh me oh my, can't find the constant: LOL

{% endcodeblock %}

### NameError

This error is called when Ruby can't find the constant and there is no `const_missing` method defined.

{% codeblock lang:ruby %}

Object::BLASTER

# => NameError: uninitialized constant BLASTER
  from (irb):8

{% endcodeblock %}

## Word about autoloading

Let's say you'd like to be flexible about your constants and load them automatically, following some naming convention?  Turns out, there is a way, it's called autoloading. 

If we were to implememt autoloading from scratch, it would be something like this: 

{% codeblock lang:ruby %}


def Object.const_missing(name)
  @looked_for ||= {}
  str_name = name.to_s
  raise "Class not found: #{name}" if @looked_for[str_name]
  @looked_for[str_name] = 1
  file = str_name.downcase
  require file
  klass = const_get(name)
  return klass if klass
  raise "Class not found: #{name}"
end

{% endcodeblock %}

Turns out, we don't have to, because autoloading is built into Ruby. We have [Kernel#autoload](http://ruby-doc.org/core-2.0.0/Kernel.html#method-i-autoload), [Module#autoload](http://ruby-doc.org/core-2.0.0/Module.html#method-i-autoload) and more sophisticated [ActiveSupport::Autoload](http://api.rubyonrails.org/classes/ActiveSupport/Autoload.html). I'm not going to cover these topics here but will try to do it in a future post.

## Ambiguity

Here comes the tricky part: what if you have multiple constants with the same name? 
Consider this example:

{% codeblock lang:ruby %}

module Insurable
  LIFE_INSURANCE_AMOUNT = 150_000
end

class Person
  LIFE_INSURANCE_AMOUNT = 50_000
end

class Pilot < Person
  INSURANCE_AMOUNT = 300_000
  include Insurable
end

puts Pilot::INSURANCE_AMOUNT
puts Pilot::LIFE_INSURANCE_AMOUNT

# => 300_000
# => 150_000

{% endcodeblock %}

Results might seem strange at first, but please remember the full search path:

{% codeblock lang:ruby %}

[Module.nesting + Module.ancestors].uniq

{% endcodeblock %}

First comes the lexical scope searching and only after the inheritance chain, where mixins are inserted between 
child and parent classes. Also, when Ruby finds a constant with a given name, it stops looking further.
