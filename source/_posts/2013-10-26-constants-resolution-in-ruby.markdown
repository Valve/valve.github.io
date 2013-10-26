---
layout: post
title: "Constants resolution in Ruby"
date: 2013-10-26 11:47
comments: true
categories: ruby
---

Ruby constants resolution has always been a somewhat confusing topic for me.
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