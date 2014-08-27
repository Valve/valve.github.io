---
layout: post
title: "JSON serialization in Rust, part 1"
date: 2014-08-25 10:08
comments: true
categories: [rust, json]
---

This post is intended for people coming from high level languages, such as
Ruby or JavaScript and who may be surprised with the complexity of
the JSON serialization infrastructure in Rust.

This is the first part of the 2 part post that deals with encoding Rust
values into JSON. Second part will deal with converting JSON strings
back into Rust values.

## Overview

JSON serialization lives in the [serialize](http://doc.rust-lang.org/serialize/index.html) crate.
It contains [json](http://doc.rust-lang.org/serialize/json/) module where low-level implementation
details live and two traits which we are interested in:
[Encodable](http://doc.rust-lang.org/serialize/trait.Encodable.html) and
[Encoder](http://doc.rust-lang.org/serialize/trait.Encoder.html).

Please note that hex and base64 modules are not relevant to JSON
serialization, so we should not pay attention to them.

<!--more -->

## Serialization

In order for a type to be JSON serializable, it must implement
[Encodable](http://doc.rust-lang.org/serialize/trait.Encodable.html)
trait. Almost all built-in types already implement it, so you can serialize
them as easily as:

{%codeblock lang:rust%}
extern crate serialize;
use serialize::json;
fn main(){
  let numeric = 3.14f64;
  let str = "Hello, world";
  println!("{}", json::encode(&numeric));
  println!("{}", json::encode(&str));
}
// 3.14
// "Hello, world"
{%endcodeblock%}
[playpen](http://is.gd/lrjNyD)

### Option

{%codeblock lang:rust%}
// assuming inside main and json in scope
let opt = Some(3.14)
println!("{}", json::encode(&opt));
let opt2: Option<f64> = None;
println!("{}", json::encode(&opt2));
// 3.14
// null
{%endcodeblock%}
[playpen](http://is.gd/uSciLe)

### Vector

{%codeblock lang:rust%}
let vec = vec!(1939i, 1945);
println!("{}", json::encode(&vec));
// [1939,1945]
{%endcodeblock%}
[playpen](http://is.gd/9e5L5L)

### HashMap
{%codeblock lang:rust%}
let mut map = HashMap::new();
map.insert("pi", 3.14f64);
map.insert("e", 2.71);
println!("{}", json::encode(&map));
// {"e":2.71,"pi":3.14}
{%endcodeblock%}
[playpen](http://is.gd/PZJOxY)

### Array
Currently Rust cannot automatically serialize fixed sized arrays to JSON.
{%codeblock lang:rust%}
let numbers = [1i, 2, 3];
json::encode(&numbers);
// failed to find an implementation of trait 
// serialize::serialize::Encodable<serialize::json::Encoder<'_>,std::io::IoError> for [int, .. 3]
// json::encode(&numbers);
{%endcodeblock%}
Array's type signature includes its length, but Rust can't be generic with array's
length. So in order to serialize an array into JSON, we'll need to use
custom serialization, which I'll explain further.

#### UPDATE: Thanks to [Reddit user ASeriesOfTubes](http://www.reddit.com/r/rust/comments/2eiyzh/json_serialization_in_rust_part_1/ck1j4ni)

It is possible to serialize an array to JSON automatically. You just
need to convert it to slice.
{%codeblock lang:rust%}
let numbers = [1i, 2, 3];
println!("{}", json::encode(&numbers.as_slice()));
// [1,2,3]
{%endcodeblock%}
[playpen](http://is.gd/1GM2WF)

### Structs

It's possible to have Rust automatically implement JSON serialization
for your structs. You'll need to adorn the struct with
`deriving(Encodable)` attribute.

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json, Encodable};
fn main(){
  let person = Person{name: "John Doe".to_string(), age: 33};
  println!("{}", json::encode(&person));
}
#[deriving(Encodable)]
struct Person {
  name: String,
  age: int
}
// {"name":"John Doe","age":33}
{%endcodeblock%}
[playpen](http://is.gd/8p8V31)

### Custom serialization

You will inevitably come to the point when Rust's supplied serialization
will not work for you. Luckily we have full control over the serialization
process. In order to serialize your type the way you want it, you will
need to implement the `Encodable` trait. Let's continue with our `Person`
struct example and change it to include a summary field.

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json, Encodable, Encoder};
fn main() {
  let person = Person{name: "John Doe".to_string(), age: 33,};
  println!("{}" , json::encode(&person));
}

struct Person {
  name: String,
  age: int,
}

impl <S: Encoder<E>, E> Encodable<S, E> for Person {
  fn encode(&self, encoder: &mut S) -> Result<(), E> {
    match *self {
      Person{name: ref p_name, age: ref p_age} => {
        encoder.emit_struct("Person", 0, |encoder| {
          try!(encoder.emit_struct_field( "age", 0u, |encoder| p_age.encode(encoder)));
          try!(encoder.emit_struct_field( "name", 1u, |encoder| p_name.encode(encoder)));
          try!(encoder.emit_struct_field( "summary", 2u, |encoder| {
            (format!("Nice person named {}, {} years of age", p_name, p_age)).encode(encoder)
          }));
          Ok(())
        })
      }
    }
  }
}
// {"age":33,"name":"John Doe","summary":"Nice person named John Doe, 33 years of age"}
{%endcodeblock%}
[playpen](http://is.gd/s1rsXL)

Let us break down this code line by line to understand what's going on.

`Line 2:` We need to bring both `Encodable` and `Encoder` traits into
scope. `Encodable` trait is for the struct to implement, to conform to
the JSON serialization interface, while the
`Encoder` is the low level workhorse of serialization, which transforms
primitive values into JSON bits and combines them all together.

`Line 8:` We no longer need to use the `#[deriving(Encodable)]`
attribute, because we're implementing the `Encodable` trait ourselves.

`Line 13:` We implement `Encodable` trait for the `Person` struct.
`Encodable` full type signature is `Encodable<S, E>` where
`S` should be an instance of `Encoder<E>`.
`E` is a type parameter for [Result<T, E>](http://doc.rust-lang.org/std/result/type.Result.html),
which our implementation returns.

`Line 14:` In order to implement the `Encodable` trait, we need to write
the `encode` method, which accepts a single `S` argument. Remember, that
`S` is an instance of `Encoder`, which is a low level JSON emitter.

`Lines 15-16:` We're destructuring (decomposing) the struct to access
its fields. To do that we use pattern matching and assign the person
fields to `p_name` and `p_age` variables.

`Line 17:` This is where JSON writing begins. We call `emit_struct` on
our encoder and pass it 3 arguments: the name of the struct, current
index and an anonymous function(aka lambda). The name of the struct
[is not used](https://github.com/rust-lang/rust/blob/0b3e43d2a47ecf4908a912c1144942e5216703ea/src/libserialize/json.rs#L500); current index
[is not used too](https://github.com/rust-lang/rust/blob/0b3e43d2a47ecf4908a912c1144942e5216703ea/src/libserialize/json.rs#L501).
What is important is the anonymous function that we're passing as the
3rd argument.
The `emit_struct` method simply writes `{`, calls the lambda and then
writes closing `}`. Why are the 1st and the 2nd arguments not used?
I think they are there to conform to the uniform style of encoder's `emit_*` methods,
but they don't make any sense when writing a JSON object.

`Lines 18-22:` This is where the body of the JSON object is written.
Each field is written with `emit_struct_field` method that accepts same
3 arguments: name, index and lambda. Name is how you want your object
field to be named, index is to correctly insert comma after each field
and the lambda's job is to return correctly escaped JSON representation of
the struct's field value. Since the built-in types already implement the
`Encodable` trait, we can safely call `encode` on integers and strings
to encode their values into JSON.

`Line 23:` To indicate the successful JSON encoding, we return unit
wrapped in Ok enum value of the Result.

`Line 24:` The line where the closing `}` of the object is written,
because lambda finishes here.

#### Serializing fixed length arrays

Now armed with the knowledge to write our own implementation of
`Encodable`, we can convert an array into JSON.

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json, Encodable, Encoder};
static BUFFER_SIZE: uint = 4;
fn main() {
  let buffer = Buffer([42i, 43, 44, 45]);
  println!("{}", json::encode(&buffer));
}

struct Buffer<T>([T,..BUFFER_SIZE]);


impl <S: Encoder<E>, T: Encodable<S,E>, E> Encodable<S, E> for Buffer<T>{
  fn encode(&self, encoder: &mut S) -> Result<(), E> {
    match *self {
      Buffer(ref data) => {
        let mut counter = 0u;
        encoder.emit_seq(BUFFER_SIZE, |encoder| {
          for i in data.iter() {
            try!(encoder.emit_seq_elt(counter, |encoder| i.encode(encoder)));
            counter += 1;
          }
          Ok(())
        })
      }
    }
  }
}
// [42,43,44,45]
{%endcodeblock%}
[playpen](http://is.gd/urSwVE)

Since rust will not allow to provide the implementation of a trait for
a type where both the trait and the type were defined in the external
crate, we need to create a tuple struct (newtype) for the array.

Overall, this implementation looks similar to the previous, except we're using
the combination of `emit_seq` + `emit_seq_elt` to emit `[` + elements +
`]`. We also keep a counter variable to correctly handle the comma.

Note that the implementation signature adds new `T` type parameter,
which is the type of the array. It can be anything that implements
`Encodable<S, E>` trait.

### deriving(Encodable)

Now you're ready to understand what happens when you use
`#[deriving(Encodable)]`. Let's use the `Person` struct example again
and compile it with `--pretty expanded` flag:

`rustc app.rs --pretty expanded`

We see the output:

{%codeblock lang:rust%}
#![feature(phase)]
#![no_std]
#![feature(globs)]
#[phase(plugin, link)]
extern crate std = "std";
extern crate serialize;
#[prelude_import]
use std::prelude::*;
use serialize::{Encodable};
struct Person {
    name: String,
    age: int,
}
#[automatically_derived]
impl <__S: ::serialize::Encoder<__E>, __E> ::serialize::Encodable<__S, __E>
     for Person {
    fn encode(&self, __arg_0: &mut __S) -> ::std::result::Result<(), __E> {
        match *self {
            Person { name: ref __self_0_0, age: ref __self_0_1 } =>
            __arg_0.emit_struct("Person", 2u, ref |_e| {
                                match _e.emit_struct_field("name", 0u,
                                                           ref |_e|
                                                               (*__self_0_0).encode(_e))
                                    {
                                    Ok(__try_var) => __try_var,
                                    Err(__try_var) => return Err(__try_var),
                                };
                                return _e.emit_struct_field("age", 1u,
                                                            ref |_e|
                                                                (*__self_0_1).encode(_e));
                            }),
        }
    }
}
{%endcodeblock%}

The implementation provided by the Rust compiler is almost identical to
ours, except that it further expanded the `try!` macros into `Err` and
`Ok` branches.

Second part of this article will explain the reverse process: how to
decode Rust objects from JSON string.

