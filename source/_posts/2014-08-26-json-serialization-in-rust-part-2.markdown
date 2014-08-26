---
layout: post
title: "JSON serialization in Rust, part 2"
date: 2014-08-26 09:09
comments: true
categories: [rust, json]
---

This post is intended for people coming from high level languages, such as
Ruby or JavaScript and who may be surprised with the complexity of
the JSON serialization infrastructure in Rust.

This is the second part of the 2 part post that deals with decoding JSON
strings into Rust values.
[First part is available here](http://valve.github.io/blog/2014/08/25/json-serialization-in-rust-part-1/).

## Overview

When working with JSON deserialization, we're interested in
[Decodable](http://doc.rust-lang.org/serialize/trait.Decodable.html) and
[Decoder](http://doc.rust-lang.org/serialize/trait.Decoder.html) traits.

As with serialization, `hex` and `base64` modules are not relevant to JSON
deserialization, so we should not pay attention to them.

<!--more -->

## Deserialization

In order for a type to be decodable from JSON, it must implement
[Decodable](http://doc.rust-lang.org/serialize/trait.Decodable.html)
trait. Almost all built-in types already implement it, so you can deserialize
them with:

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json};

fn main(){
  let numeric_s = "3.14";
  let greeting = "\"Hello, world\"";
  let pi: f64 = json::decode(numeric_s).unwrap();
  println!("{}", pi);
  let decoded_greeting: String = json::decode(greeting).unwrap();
  println!("{}", decoded_greeting);
}
// 3.14
// Hello, world
{%endcodeblock%}
[playpen](http://is.gd/N3p1IE)

### Option

Deserializing to Option is somewhat redundant, because `json::decode`
[returns](https://github.com/rust-lang/rust/blob/5fb2dfaa200f2cb32e77c54ae8a5e0f4823b65c8/src/libserialize/json.rs#L288)
[DecodeResult](http://doc.rust-lang.org/serialize/json/type.DecodeResult.html), which is a type alias
for a regular result. That means you can pattern match on DecodeResult
and handle potential failure.

### Vector
{%codeblock lang:rust%}
// assuming in main with json in scope
let raw_json = "[42, 43, 44, 45]";
let vec: Vec<i32> = json::decode(raw_json).unwrap();
println!("{}", vec);
// [42, 43, 44, 56]
{%endcodeblock%}
[playpen](http://is.gd/z5cLqu)

### Tuple

Decoding JSON to a tuple is identical to vector, just specify the
correct type:
{%codeblock lang:rust%}
// assuming in main with json in scope
let raw_json = "[42, 43]";
let tup: (i32, i32) = json::decode(raw_json).unwrap();
println!("{}", tup);
// (42, 43)
{%endcodeblock%}
[playpen](http://is.gd/KTvqSC)

### HashMap

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json};
use std::collections::HashMap;

fn main(){
  let raw_json = "{\"e\":2.71,\"pi\":3.14}";
  let map: HashMap<String, f64> = json::decode(raw_json).unwrap();
  println!("{}", map);
}
// {pi: 3.14, e: 2.71}
{%endcodeblock%}
[playpen](http://is.gd/f2Smhd)

### Array

As with serializing, Rust cannot automatically deserialize JSON string
into a fixed-length array. The reason for this is the same: arrays' type
signature contain length as part of the type, but Rust currently (and
most likely not until after v1.0) can't be generic over array's length.

{%codeblock lang:rust%}
let raw_json = "[42, 43]";
let arr: [i32,..2] = json::decode(raw_json).unwrap();
println!("{}", arr);
// failed to find an implementation of trait
// serialize::serialize::Decodable<serialize::json::Decoder,serialize::json::DecoderError> 
// for [i32, .. 2]
{%endcodeblock%}
[playpen](http://is.gd/c8N3CV)

To remedy this, we will use custom decoding, as we did with custom array
encoding in part 1.
I'll show an example of this below.

### Structs

As with serialization, it's possible to have Rust deserialize structs
automatically for you. You will need to add the
`#[deriving(Decodable)]` attribute to your struct:

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json};

fn main(){
  let raw_json = "{\"name\":\"John Doe\",\"age\":33}";
  let person: Person = json::decode(raw_json).unwrap();
  println!("{}", person);
}

#[deriving(Encodable, Decodable, Show)]
struct Person {
  name: String,
  age: int
}
// Person { name: John Doe, age: 33 }
{%endcodeblock%}
[playpen](http://is.gd/SdeDWO)

Note that I'm using 3 deriving trait implementations for a struct:
`Encodable`, `Decodable` and `Show`. This is to make my struct fully
JSON (de)serializable and printable automatically.

### Custom deserialization

This is probably the cornerstone of the JSON infrastructure.
In real life you often cannot control the shape of the JSON that comes to
you, so you must be able to convert arbitrary JSON strings into your
objects. Luckily, Rust decoding capabilities will help us here.

Let us continue with our `Person` struct example and deserialize the
object from a complex JSON where our object is in the `data`
key. The example might be contrived, but it serves the demo purpose.

To make a type JSON deserializable we need to implement the `Decodable`
trait.

{%codeblock lang:rust%}
extern crate serialize;
use serialize::{json, Decodable, Decoder};

fn main(){
  let raw_json = r#"{"type": "Person", "data": {"name": "John Doe", "age": 33}}"#;
  let person: Person = json::decode(raw_json).unwrap();
  println!("{}", person);
}

#[deriving(Show)]
struct Person {
  name: String,
  age: int
}

impl<S: Decoder<E>, E> Decodable<S, E> for Person {
  fn decode(decoder: &mut S) -> Result<Person, E> {
    decoder.read_struct("root", 0, |decoder| {
      decoder.read_struct_field("data", 0, |decoder| {
         Ok(Person{
          name: try!(decoder.read_struct_field("name", 0, |decoder| Decodable::decode(decoder))),
          age: try!(decoder.read_struct_field("age", 0, |decoder| Decodable::decode(decoder)))
        })
      })
    })
  }
}
// Person { name: John Doe, age: 33 }
{%endcodeblock%}
[playpen](http://is.gd/TKPROs)

Let's break down the code line by line to see what's going on here.

`Line 2:` We need to bring both `Decodable` and `Decoder` traits into
scope. `Decodable` trait is for the struct to implement, to conform to
the JSON deserialization interface, while the
`Decoder` is the low level workhorse of deserialization, which tokenizes
and parses the JSON string to convert it to Rust values later.

`Line 5:` I'm using a raw string literal to avoid escaping double quotes.

`Line 6:` The line where I'm decoding JSON string into an instance of
`Person` struct. Note that I need to type-annotate the variable when
decoding it.

`Line 10:` We no longer need to use the `#[deriving(Decodable)]`
attribute, because we implement the `Decodable` trait ourselves.

`Line 16:` This is the `Decodable` implementation. It is very similar to
the `Encodable` implementation from part 1 with the exception of `S` being
type restricted to `Decoder` trait now.

`Line 17:` Two differences from `encode` method counterpart:
we're no longer accepting `&self` as the first argument,
because `decode` is an associated function, rather than a method.
The analogy is class methods in Ruby or static methods
in Java. Second difference is the return type. It is now `Result<Person,
E>`.

`Line 18:` This is where the parsing starts. We do actual parsing with
`read_*` family of methods on `Decoder` instance. Here we're reading the
top-level struct with `read_struct` method. First argument is the name
of the sturct(not used), second is the length (not applicable). The third argument is an
instance of anonymous function (lambda). Why are the first and the
second arguments not used?
I think this is because the entire family of `read_*` methods of
`Encoder` strives to be uniform and thus a unified set of arguments
is used, even when the encoder does not need them.

You can think of the `read_struct` call as "opening" the top-level JSON
object to be able to move inside to read actual values. The lambda is
where we descend and continue with reading.

`Line 19:` The object we're trying to read is in the `data` field, so
we're reading it on this line with `read_struct_field` method.
This time the first argument is necessary, because it tells the decoder
the actual name of the field. 3rd argument is the lambda again to
descend further into the the object in the `data` field.

`Lines 20-21:` Field reading happens here. By this time the parser
has reached the contents of the `data` object so we can now just read the
fields we're interested in one-by-one. We're using `read_struct_field`
again, passing it the name of the field and the index(not used).
The third argument is the value of the field, correctly decoded from
JSON representation. Since all primitive values in Rust already
implement the `Decodable` trait, we can safely call `Decodable::decode`
on them to deserialize them as the `Person` struct fields.

#### Deserializing fixed length arrays

As in part 1, let's use this knowledge to deserialize a fixed-length
array from JSON.

{%codeblock lang:rust%}
extern crate serialize;
use std::default::Default;
use serialize::{json, Decodable, Decoder};

static BUFFER_SIZE: uint = 4;

fn main(){
  let raw_json = "[42, 43, 44, 45]";
  let buf: Buffer<i32> = json::decode(raw_json).unwrap();
  let Buffer(arr) = buf;
  for i in arr.iter() {
    println!("{}", i);
  }
}

struct Buffer<T>([T,..BUFFER_SIZE]);

impl<S: Decoder<E>, T: Default+Copy+Decodable<S, E>, E> Decodable<S, E> for Buffer<T> {
  fn decode(decoder: &mut S) -> Result<Buffer<T>, E> {
    decoder.read_seq(|decoder, len| {
      if len != BUFFER_SIZE {
        return Err(decoder.error(format!("Expecting array of length: {}, but found {}", BUFFER_SIZE, len).as_slice()));
      }
      let mut arr: [T,..BUFFER_SIZE] = [Default::default(),..BUFFER_SIZE];
      for (i, val) in arr.mut_iter().enumerate() {
        *val = try!(decoder.read_seq_elt(i, Decodable::decode));
      }
      Ok(Buffer(arr))
    })
  }
}
// 42
// 43
// 44
// 45
{%endcodeblock%}
[playpen](http://is.gd/zl076C)

Since rust will not allow to provide the implementation of a trait for
a type where both the trait and the type were defined in the external
crate, we need to create a tuple struct (newtype) for the array.

Overall, this implementation looks similar to the previous, but there are
nuances I'd like to point out.

`Line 18:` Note that the implementation signature adds new `T` type parameter,
which is the type of the array. It can be anything that implements
`Default`+`Copy`+`Encodable<S, E>` traits.
`Default` is to be able to fill array with default values (line 24).
`Copy` is to be able to copy the default values
into the new array, while the `Decodable<S, E>` is to be able to decode
the array elements from JSON.

`Lines 22-24:` Here I'm checking if the array we're about to decode contains
exactly the number of elements we expect. If not, I exit early with an
error.

`Lines 26-28:` Here I'm iterating the array, obtaining mutable
references to its elements and filling them from JSON, using
`decoder::read_seq_elt`.

`Line 29:` Here I'm returning the result wrapped in `Buffer` newtype.

### deriving(Decodable)

As in part 1, let's look at the expanded implementation of the
`#[deriving(Decodable)]` attribute.
Let's use the `Person` struct example again
and compile it with `--pretty expanded` flag:

`rustc app.rs --pretty expanded`

The output:

{%codeblock lang:rust%}
#![feature(phase)]
#![no_std]
#![feature(globs)]
#[phase(plugin, link)]
extern crate std = "std";
extern crate rt = "native";
extern crate serialize;
#[prelude_import]
use std::prelude::*;
use serialize::{Decodable};
struct Person {
    name: String,
    age: int,
}
#[automatically_derived]
impl <__D: ::serialize::Decoder<__E>, __E> ::serialize::Decodable<__D, __E>
     for Person {
    fn decode(__arg_0: &mut __D) -> ::std::result::Result<Person, __E> {
        __arg_0.read_struct("Person", 2u,
                            ref |_d|
        ::std::result::Ok(Person{name:
           match _d.read_struct_field("name",
                                      0u,
                                      ref
                                          |_d|
                                          ::serialize::Decodable::decode(_d))
               {
               Ok(__try_var)
               => __try_var,
               Err(__try_var)
               =>
               return Err(__try_var),
           },
       age:
           match _d.read_struct_field("age",
                                      1u,
                                      ref
                                          |_d|
                                          ::serialize::Decodable::decode(_d))
               {
               Ok(__try_var)
               => __try_var,
               Err(__try_var)
               =>
               return Err(__try_var),
           },}))
    }
}
{%endcodeblock%}

The output is very similar to the manual deserialization code we
saw earlier, except that compiler further expanded the `try!` macros into `Err` and
`Ok` branches.

### Further reading

There is a convenience `ToJson`
[trait](https://github.com/rust-lang/rust/blob/5fb2dfaa200f2cb32e77c54ae8a5e0f4823b65c8/src/libserialize/json.rs#L2222) that allows implementors to quickly convert itself into JSON using
intermediate [Json enum](https://github.com/rust-lang/rust/blob/5fb2dfaa200f2cb32e77c54ae8a5e0f4823b65c8/src/libserialize/json.rs#L211) representation, but I recommend using it only for small and relatively simple data structures.
