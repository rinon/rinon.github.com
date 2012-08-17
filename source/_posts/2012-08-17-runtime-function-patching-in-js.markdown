---
layout: post
title: "Runtime function patching in JS"
date: 2012-08-17 15:00
comments: true
categories: JavaScript
---

Have you ever needed to create a stub function which patched itself
with the real function at runtime? As part of some research I've been
working on lately, I found that I needed to do exactly this. Just to
make things more challenging this needed to be automated for *any*
valid JS function, no matter how crazy and convoluted. This post will
explore how I went about this and the rewriting I ended up with. I'll
try to stick fairly close to how my function patching code actually
evolved, but this should certainly not be treated as a history.

*Disclaimer:* This is still a work in progress, and although it's been
tested with fairly large and comprehensive code bases, there may
certainly be edge cases I haven't thought of. Please comment or send
me a message if you find any!

### Background

I'm currently working on a JavaScript source-to-source compiler which
transforms any arbitrary function into a string and defers the
execution and parsing of the function until it gets called (if
ever). To do this, the compiler needs to rewrite a function into a
stub which fetches and parses the original code, and then replaces
itself with the real function.

### Starting out

Let's trace an example of rewriting a simple function. I'll be using
human readable variable names, but in a compiler all generated
variable names would need to be checked against current valid variable
names and/or be namespaced to not conflict with any existing variable
names.

Here's a simple example function to start out with:

{% codeblock lang:javascript %}
function foo() {
  console.log('Hello, world!');
}

// and actually execute the function
foo();
{% endcodeblock %}

It turns out that in JavaScript you can assign a value to a function
name from inside that function. So, I started by simply replacing the
original function with the following stub:

{% codeblock lang:javascript %}
var fooStr = "function() {console.log('Hello, world!');}";

function foo() {
  foo = loadFunction(fooStr);
  foo();
}
{% endcodeblock %}

Note: We can do an analogous rewrite to anonymous functions assigned
to variables. Awesome, we just need to define `loadFunction` and we're
done, right? Shouldn't be too hard... Let's give it a try:

{% codeblock lang:javascript %}
function loadFunction(fnStr) {
  return eval(fnStr);
}
{% endcodeblock %}

There, that should just about do it... What's that you say?  `eval()`
is complaining about not being able to parse the function?  Oh, let's
see, it seems that the anonymous function needs to be wrapped in
parens so it can be parsed as a StatementExpression! Ok, fine:

{% codeblock lang:javascript %}
var fooStr = "(function() {console.log('Hello, world!');})";
{% endcodeblock %}

Awesome, it works now!

One thing to note here -- `loadFunction` needs to be in the same frame
(function scope) as the original function so that the function created
with `eval` closes over the same variables as the original function
would have. This means that nested functions will need a different
`loadFunction` at each nesting level containing a rewritten function
so the `eval` occurs in the correct scope.

### Patching references

Now, we're not done yet... What if some other part of the code took a
reference to `foo` before it executed? Something simple like `var
fooAlias = foo;` would do this.  Now, `fooAlias` will get a reference
to the stub, and every time `fooAlias` was called, the function would
have to be recreated. This is, of course, horribly bad for
performance! While we can't replace references to the stub, we can
make the stub into a simple trampoline that stores the loaded function
and jumps to it if available. Although there are probably other ways
to do this, I chose to set the function string variable (`fooStr` in
this case) to null, and check to make sure it was a valid string
before the `eval`. With this change, the code now looks like:

{% codeblock lang:javascript %}
function loadFunction(fnStr, fn) {
  if (typeof fnStr === 'string') {
    return eval(fnStr);
  } else {
    return fn;
  }
}

function foo() {
  foo = loadFunction(fooStr, foo);
  fooStr = null;
  foo();
}

var fooAlias = foo;

fooAlias();
fooAlias();
{% endcodeblock %}

The first time `fooAlias` is called, `fooStr` will be a valid string
and the function will be created as it should. The second call will
still result in a call to `loadFunction`, but this will immediately
return the replaced version of `foo`, without recreating it.

### Redefinition

Well, now this is all fine and dandy, but does it work for all cases?
Unfortunately, as you may have guessed, not yet. First of all, what
happens if the code redefines `foo` after we rewrite it? (I ran into
this while trying to compile
[BananaBread](https://github.com/kripken/BananaBread), where a
function is redefined to extend it with a bit of new
functionality). For example:

{% codeblock lang:javascript %}
function foo() {
  console.log('foo');
}

var oldFoo = foo;

foo = function() {
  console.log('In the new foo');
};

oldFoo();
foo();
{% endcodeblock %}

should output:

    foo
    now in the new foo

Using the current transformation, we would rewrite this code to:

{% codeblock lang:javascript %}
var fooStr1 = '(function(){console.log("foo")})';
var fooStr2 = '(function(){console.log("In the new foo")})';

function foo() {
  foo = loadFunction(fooStr1, foo);
  fooStr1 = null;
  foo();
}

var oldFoo = foo;

var foo = function () {
  foo = loadFunction(fooStr2, foo);
  fooStr2 = null;
  foo();
};

oldFoo();
foo();
{% endcodeblock %}

but this outputs:

    foo
    foo

Wait a second! That's not what we expected! What happened? Well, if
you trace carefully, you will see that the call to `oldFoo()` on line
18 overwrites `foo` with the version of `foo` generated from
`fooStr1`.  This is then called as `foo()` on line 19 and the old
version is executed, rather than the correctly redefined version.

We can avoid this by overwriting the stub with the generated function
*only* when it has not been changed elsewhere. This is easily done by
checking that `foo === arguments.callee` before overwriting `foo` (be
careful here: `arguments.callee` is not the same as `this` in the
browser, as one might expect -- see
[this MDN doc](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Operators/this)
for more info).

With this change, the rewritten code now looks like:

{% codeblock lang:javascript %}
var fooStr1 = '(function(){console.log("foo")})';
var fooStr2 = '(function(){console.log("In the new foo")})';

function foo() {
  var temp = loadFunction(fooStr1, foo);
  fooStr1 = null;
  if (foo === arguments.callee)
    foo = temp;
  temp();
}

var oldFoo = foo;

var foo = function () {
  temp = loadFunction(fooStr2, foo);
  fooStr2 = null;
  if (foo === arguments.callee)
    foo = temp;
  temp();
};

oldFoo();
foo();
{% endcodeblock %}

This now works as expected. Unfortunately `loadFunction` will still get called
every time `oldFoo` is called, but fixing this would add length to the
stub, which I'd like to avoid.

Unfortunately we also need to keep track of each stub (and
replacement) so that `loadFunction` works correctly. Without this, a
second call to `oldFoo` which in turn calls `loadFunction`, returning
`foo`, will return the wrong version of `foo`! If that seems a bit
convoluted, it's probably because it is. Hopefully tracing through the
code will help:



{% codeblock lang:javascript %}
var generated1 = foo, generated2;
var fooStr1 = '(function(){console.log("foo")})';
var fooStr2 = '(function(){console.log("In the new foo")})';

function foo() {
  generated1 = loadFunction(fooStr1, generated1);
  fooStr1 = null;
  if (foo === arguments.callee)
    foo = generated1;
  generated1();
}

var oldFoo = foo;

var foo = function () {
  generated2 = loadFunction(fooStr2, generated2);
  fooStr2 = null;
  if (foo === arguments.callee)
    foo = generated2;
  generated2();
}, generated2 = foo;

oldFoo();
foo();
{% endcodeblock %}

This stores the stubs in `generated1` and `generated2` respectively,
and then overwrites these vars when the function is loaded, regardless
of whether we can replace the `foo` name itself. Pay special attention
to where these vars were assigned to. `generated1` must be assigned at
the top of the function because function declarations are lifted to
the top of their frame in JavaScript, and we cannot assign to
`generated2` until after `foo` is redefined.

All that's left now for the stub is to properly pass arguments and
context to the loaded function by using `.apply()`:

{% codeblock lang:javascript %}
function foo() {
  generated1 = loadFunction(fooStr1, generated1);
  fooStr1 = null;
  if (foo === arguments.callee)
    foo = generated1;
  generated1.apply(this, arguments);
}
{% endcodeblock %}

### Properties

Finally, there's one last piece of the puzzle -- copying over any
attributes that should be attached to the function being loaded, but
that instead were attached to the stub. For example:

{% codeblock lang:javascript %}
function foo() {
  generated = loadFunction(fooStr, generated);
  fooStr = null;
  if (foo === arguments.callee)
    foo = generated;
  generated.apply(this, arguments);
}

foo.prototype.prototypeMethod = function() {
}

foo.objectMethod = function() {
}
{% endcodeblock %}

As you can see, `prototypeMethod` and `objectMethod` were attached to
the stub prototype and object. When `foo` is loaded and the stub
replaced, these methods disappear! We need to extend `loadFunction` to
copy any properties over to the loaded function like so:

{% codeblock lang:javascript %}
function loadFunction(fnStr, fn) {
  if (typeof fnStr === 'string') {
    var temp = eval(fnStr);
    temp.prototype = fn.prototype;
    for (var prop in fn) {
      temp[prop] = fn[prop];
    }
    return temp;
  } else {
    return fn;
  }
}
{% endcodeblock %}

### Putting it all together

Here's my final version of function loading and patching for a typical
function:

{% codeblock lang:javascript %}
var generated = foo;

function loadFunction(fnStr, fn) {
  if (typeof fnStr === 'string') {
    var temp = eval(fnStr);
    temp.prototype = fn.prototype;
    for (var prop in fn) {
      temp[prop] = fn[prop];
    }
    return temp;
  } else {
    return fn;
  }
}

var fooStr = '(function(){console.log("foo")})';


function foo() {
  generated = loadFunction(fooStr, generated);
  fooStr = null;
  if (foo === arguments.callee)
    foo = generated;
  generated.apply(this, arguments);
}

foo();
{% endcodeblock %}

This covers every edge case that I could think of, and should work to
patch a function at runtime (as well as possible).
