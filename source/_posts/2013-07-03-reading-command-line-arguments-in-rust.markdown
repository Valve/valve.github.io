---
layout: post
title: "reading command line arguments in rust"
date: 2013-07-03 13:31
comments: true
categories: rust
---

Since all rust hello world tutorials usually start like this:
{% codeblock lang:rust %}
fn main(){
  println("Hello, world");
}
{% endcodeblock %}

It's unclear where to take command line arguments. 
The solution:
{% codeblock lang:rust %}
use std::os;

fn main(){
  let args: ~[~str] = os::args();
  println(args.to_str());
}
{% endcodeblock %}

This will give you a vector with strings where first string will be the app being run and the rest is the 
arguments provided.

`rust-0.7`

