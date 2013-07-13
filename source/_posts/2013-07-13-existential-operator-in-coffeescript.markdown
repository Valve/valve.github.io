---
layout: post
title: "existential operator in CoffeeScript"
date: 2013-07-13 12:39
comments: true
categories: coffeescript
---

Existential operator `?` can be used in three useful ways in CoffeeScript.

<!--more-->

### Checking the existence of a variable

In JavaScript there is no built-in way of checking the existence of a variable.
You can try testing the existence with `if(variable){...}` but it won't work in these cases:

{% codeblock lang:javascript %}
if(0){
  console.log('this will not print');
}

if(""){
  console.log('this will not print');
}

if(false){
  console.log('this will not print');
}

// abc was never declared
if(abc){
  //ReferenceError: abc is not defined
}
{% endcodeblock %}

The correct way to test if variable was both declared and initialized:

{% codeblock lang:javascript %}
if(typeof variable !== 'undefined' && variable !== null){
  console.log('variable was declared and initialized with a value');
}
{% endcodeblock %}

You may be tempted to use direct comparison with `undefined`:

{% codeblock lang:javascript %}
if(variable !== undefined ...
{% endcodeblock %}

But this is asking for trouble because in EcmaScript 3 (all older browsers, such as IE 6-8)  `undefined` can be overwritten:

{% codeblock lang:javascript %}
window.undefined = 'pancakes';

if('pancakes' === undefined){
  console.log('This will print, if you are unfortunate enough to use IE 6-8');
}
{% endcodeblock %}

[All ES5 compatible browsers](http://kangax.github.io/es5-compat-table/) have `undefined` immutable, i.e. it can't be changed, but it's better to play it safe.

CoffeeScript has a syntactic shortcut for testing existence:

{% codeblock lang:coffeescript CoffeeScript %}
if variable?
  console.log('variable is both declared and initialized with a non-null value');
{% endcodeblock %}

This code will be transpiled to:

{% codeblock lang:javascript %}
if (typeof variable !== "undefined" && variable !== null) {
  console.log('If variable was both declared and initialized with a non-null value');
}
{% endcodeblock %}

### Conditional assignment

What if you want to initialize a variable only if it has not been already initialized?
In JavaScript you'd usually do something like:

{% codeblock lang:javascript %}

function getUserLocale(){
  if(this.locale == null){
    this.locale = DB.getLocaleByUser(User.current);
  }
  return this.locale;
}

{% endcodeblock %}

This technique caches the result of an expensive computation or database query in a variable.
In above example, all subsequent `getUserLocale` function calls will not query the database.
The important part is comparing with `null` using `==`, rather than `===`, because `==` will evaluate to `true` if variable is either `undefined` or `null`.

Such _cache on first call_ pattern is widely used in many programming languages, but CoffeeScript has a special syntax for it:

{% codeblock lang:coffeescript CoffeeScript %}

getUserLocale = ->
  @locale ?= DB.getLocaleByUser(User.current)

{% endcodeblock %}

`?=` is the operator that performs conditional assignment.

This will be transpiled to roughly the same JS as above, using ternary operator:

{% codeblock lang:javascript %}
getUserLocale = function() {
  return this.locale != null ? this.locale : this.locale = DB.getLocaleByUser(User.current);
};
{% endcodeblock %}

What if you try to conditionally assign an undeclared variable? If you try this:

{% codeblock lang:coffeescript CoffeeScript %}
# abc is not declared 
abc ?= 99
{% endcodeblock %}

This will result in a compile-time error:
` the variable "abc" can't be assigned with ?= because it has not been declared before`.
This compile-time checking is very helpful, because it prevents a `ReferenceError` at run time.

If you know Ruby, `||=` is the same thing there.

### Safe property / function chaining

Chaining function calls is a great way to write terse yet fluent code.
A good example is working with jQuery:
{%codeblock lang:javascript %}

$('#header').css('color', '#fadfad').show('slow').off();
{% endcodeblock %}

This is made possible because these jQuery functions return a reference to `this`.
But what if one of the function returns `null` or `undefined`?

{%codeblock lang:javascript %}
var zip = User.current.address.zip
{%endcodeblock%}

If current user's address is `null` or `undefined`, the `.zip` property call will result in `TypeError`.

A simple but ugly solution would be to use a lot of `if` checks:

{%codeblock lang:javascript%}
var zip = null;
if(User.current != null && User.current.address != null){
  // only now this is safe
  zip = User.current.address.zip;
}
{%endcodeblock %}

But this can quickly get out of hand with deep nesting.

CoffeeScript has a safe way of accessing long property chains using `?.` variant of existential operator:
{%codeblock lang:coffeescript CoffeeScript %}
zip = User.current?.address?.zip
{%endcodeblock%}

This will either soak up the `null` or `undefined` references and safely return `undefined` or
return the final property value. In our case the generated code looks like this:

{%codeblock lang:javascript %}
var zip, _ref;

zip = (_ref = User.current) != null ? _ref.address.zip : void 0;
{%endcodeblock %}

What is this weird `void 0` thing? This is to fight the pre AS5 `undefined` mutability I referred to earlier.
[JavaScript defines void as a unary operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void) that returns `undefined` for any argument. In other words, CoffeeScript compiler uses a set of nested ternary operators to safely return either last property value or `undefined` with `void 0`.

Calling a function safely works similarly:

{% codeblock lang:coffeescript CoffeeScript %}
noSuchFunction?()
{% endcodeblock %}

This transpiles to:
{%codeblock lang:javascript %}
if (typeof noSuchFunction === 'function') {
  noSuchFunction();
}
{%endcodeblock %}

Key thing to take away here is that CoffeeScript first tests that callable function is defined and is a function.
It does this using `typeof bla === 'function'`. The function is called only if it is defined.

Safe function invocation can be chained as well with other function or property calls:

{% codeblock lang:coffeescript example from coffeescript.org %}
lottery.drawWinner?().address?.zip
{% endcodeblock %}

This transpiles to:
{%codeblock lang:javascript %}
var zip, _ref;
zip = typeof lottery.drawWinner === "function" 
  ? 
    (_ref = lottery.drawWinner().address) != null ? _ref.zipcode : void 0 
  : 
    void 0;
{%endcodeblock%}

Drawing analogy with Ruby-on-Rails ActiveSupport, the safe chaining can be seen as [try](http://api.rubyonrails.org/classes/NilClass.html#function-i-try) method.

#### Conclusion:

CoffeeScript existential operator is a useful tool to cut down the verbosity of JavaScript when dealing with existence and null checks.
It also can be used to shield inexperienced JavaScript developers from JavaScript bad parts.
