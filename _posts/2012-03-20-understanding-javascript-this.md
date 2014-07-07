---
layout: post
title: Understanding the "this" keyword in JavaScript
slug: understanding-this-javascript
---

Many people get tripped up by the `this` keyword in JavaScript. I think the
confusion comes from people reasonably expecting `this` to work like "this"
does in Java or the way people use "self" in Python. Although `this` is
sometimes used to similar effect, it's nothing like "this" in Java or other
languages. And while it's a little harder to understand, its behavior isn't
magic. In fact, `this` follows a relatively small set of simple rules. This
post is an explanation of those rules.

But first, I want to give some terminology and information about JavaScript
runtime environments which will hopefully help you develop a mental model to
better understand `this`.<sup><a href="#mightbewrong">[1]</a></sup><a
name="mightbewronglink"></a>

Execution contexts
==================

Every line of JavaScript code is run in an "execution context." The JavaScript
runtime environment maintains a stack of these contexts, and the top execution
context on this stack is the one that's actively running.

There are three types of executable code: _Global code_, _function code_, and
_eval code_. Roughly speaking, *global code* is code at the top level of your
program that's not inside any functions, *function code* is code that's inside
the body of a function, and *eval code* is global code evaluated by a call to
`eval`.

The object that `this` refers to is redetermined every time control enters
a new execution context and remains fixed until control shifts to a different
context. The value of `this` is dependent upon two things: The type of code
being executed (i.e., global, function, or eval) and the caller of that code.

The global object
=================

All JavaScript runtimes have a unique object called the _global object_. Its
properties include built-in objects like `Math` and `String`, as well as extra
properties provided by the host environment.

In browsers, the global object is the `window` object. In Node.js, it's just
called the "global object." (I'm going to assume you're running code in
a browser, however, everything I say should apply to Node.js as well.)

Determining the value of this
=============================

The first rule is simple: _`this` refers to the global object in all global
code._ Since all programs start by executing global code, and `this` is fixed
inside of a given execution context, we know that, by default, `this` is the
global object.

What happens when control shifts to a new execution context? There are only
three cases where the value of `this` changes: method invocations, functions
called with the `new` operator, and functions called using `call` and `apply`.
I'll explain each of these in turn.


Method invocations
==================

If we call a function as a property of an object using either dot (i.e.,
`obj.foo()`) or bracket (i.e., `obj["foo"]()`) notation, `this` will refer to
the parent object in the body of the function:

{% highlight javascript %}
var counter = {
  val: 0,
  increment: function () {
    this.val += 1;
  }
};
counter.increment();
console.log(counter.val); // 1
counter['increment']();
console.log(counter.val); // 2
{% endhighlight %}

This is our second rule: *`this` refers to the parent object inside function
code if the function is called as a property of the parent.*

Note that if we call the same function directly, that is, not as a property of
the parent object, `this` does *not* refer to the `counter` object:

{% highlight javascript %}
var inc = counter.increment;
inc();
console.log(counter.val); // 2
console.log(val); // NaN 
{% endhighlight %}

Here, `this` was not changed when `inc` was called, so it still referred to the
global object. When `inc` ran 

{% highlight javascript %}
this.val += 1;
{% endhighlight %}

it was effectively running:

{% highlight javascript %}
window.val += 1;
{% endhighlight %}

`window.val` is `undefined`, and adding `1` to `undefined` yields `NaN`.

The new operator
==================

Any JavaScript function can be used as a constructor function with `new`. The
`new` operator creates a new object and sets `this` to the new object inside
the function it was called with. For example:

{% highlight javascript %}
function F (v) {
  this.val = v;
}
var f = new F("Woohoo!");
console.log(f.val); // Woohoo!
console.log(val); // ReferenceError
{% endhighlight %}

This leads to our third rule: *`this` in function code invoked using the `new`
operator refers to the newly created object.*

Note that there's nothing special about `F`. If we call it _without_ using `new`,
`this` will refer to the global object:

{% highlight javascript %}
var f = F("Oops!");
console.log(f.val); // undefined
console.log(val); // Oops!
{% endhighlight %}

In this case, `F("Oops!")` is a regular function call, and `this` doesn't
get set to a new object, because no new object is created since the `new`
operator isn't used. `this` remains set to the global object.

Call and apply
==============

All JavaScript functions have two methods, `call` and `apply`, which let you
call functions and explicitly set the value of `this`. The `apply` method takes
two arguments: an object to set `this` to, and an (optional) array of arguments
to pass to the function:

{% highlight javascript %}
var add = function (x, y) {
      this.val = x + y;
    },
    obj = {
      val: 0
    };
add.apply(obj, [2, 8]);
console.log(obj.val); // 10
{% endhighlight %}

The `call` method works exactly the same as `apply`, but you pass the arguments
individually rather than in an array:

{% highlight javascript %}
add.call(obj, 2, 8);
console.log(obj.val); // 10
{% endhighlight %}

This is our fourth rule: *`this` is set to the first argument passed to `call`
or `apply` inside function code when that function is called with either `call`
or `apply`.*

Summary
========

That's it! You can figure out what object `this` refers to by following a few simple rules:
     
 - By default, `this` refers to the global object.
 - When a function is called as a property on a parent object, `this` refers to
   the parent object inside that function.
 - When a function is called with the `new` operator, `this` refers to the
   newly created object inside that function.
 - When a function is called using `call` or `apply`, `this` refers to the
   first argument passed to `call` or `apply`. If the first argument is `null`
   or not an object, `this` refers to the global object.

If you understand and follow those four rules, you will always know what `this` is.

...Addendum: Eval breaks all the rules
======================================

Remember when I said that code evaluated inside `eval` is its own type of
executable code? Well, that's true, and it means that the rules for determining
what `this` refers to inside of eval code are a little more complex.

As a first pass, you might think that `this` directly inside eval refers to the
same object as it does in `eval`'s caller's context. For example:

{% highlight javascript %}
var obj = {
  val: 0,
  func: function() { 
    eval("console.log(this.val)");
  }
};
obj.func(); // 0
{% endhighlight %}

That works likely as you expect it to. However, there are many cases with
`eval` where `this` will probably not work as you expect:

{% highlight javascript %}
eval.call({val: 0}, "console.log(this.val)"); // depends on browser
{% endhighlight %}

The output of the above code depends on your browser. If your JavaScript
runtime implements [ECMAScript
5](http://www.ecma-international.org/publications/standards/Ecma-262.htm),
`this` will refer to the global object and the above should print `undefined`,
because it's an "indirect" call of `eval`. That's what the latest versions of
Chrome and Firefox do. Safari 5.1 actually throws an error (_"The 'this' value
passed to eval must be the global object from which eval originated"_), which
is kosher according to [ECMAScript
3](http://www.ecma-international.org/publications/files/ECMA-ST-ARCH/ECMA-262,%203rd%20edition,%20December%201999.pdf). 

If you want know how `this` works with `eval`, you should read Juriy Zaytsev's
excellent, ["Global eval. What are the
options?"](http://perfectionkills.com/global-eval-what-are-the-options) You'll
learn more about `eval`, execution contexts, and direct vs. indirect calls than
you probably ever wanted to know.

In general, you should just avoid using `eval`, in which case the
simple rules about `this` given previously will always hold true.

<ol class="footnote">
<li>
  <p><a name="mightbewrong"></a>This is a somewhat simplified view of things to make developing a basic mental model easier. For a more detailed explanation, read the section starting at page 37 of the <a href="http://www.ecma-international.org/publications/files/ECMA-ST-ARCH/ECMA-262,%203rd%20edition,%20December%201999.pdf">ECMAScript Language Specification</a>. I've based this post in large part off of that document, however, my understanding of it is far from perfect or complete, so please let me know (<code>me &#64; nicholasbs.net</code>) if I've misunderstood or misrepresented anything.<a href="#mightbewronglink">↩</a></p>
</li>
</ol>
