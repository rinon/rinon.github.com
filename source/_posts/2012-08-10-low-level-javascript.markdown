---
layout: post
title: "Low-Level JavaScript"
date: 2012-08-10 11:14
comments: true
categories: JavaScript
---

This summer I'm taking a short break from grad school and working as a
research intern at Mozilla. Awesome experience so far, and it's great
to be back to working on open source projects! I don't think I've ever
cloned so many github repos in such a short time.

Ever wanted to have manual memory allocation in JavaScript? How about
types? One of the projects that I'm helping out with is
[LLJS](http://lljs.org), a typed dialect of JavaScript which resembles
C/C++. It features manual memory allocation when you want it,
co-existing alongside JavaScript's normal garbage-collected object
system. The goal here is to allow programmers to write fast JS code
with a familiar syntax. Definitely be sure to check out the
interactive demos to see examples of LLJS in action.

### What's new with LLJS?

We recently added arrays, with both statically stack-allocated and
dynamically heap-allocated variants. These resemble C and C++ array
syntax, respectively. I also added `union` types, to complement our
existing `struct` types. These behave precisely as you would expect
from C. Recently I added syntax for defining structs and unions inside
other structs or unions, which significantly simplifies writings
structs.

I also worked on optimizing our malloc implementation, which is much
faster now. We use a naive malloc algorithm from the K&R book, but
since we don't actually need to worry about paging, I modified this so
that all memory is on a single page. This cut out function calls for
allocating new pages, speeding allocation up quite a bit.

My mentor here at Mozilla, Michael Bebenita, added functions and
optional constructors to structs, which are beginning to look a lot
like C++ structs rather than C structs.

Finally, perhaps the most exciting new thing in LLJS is
[memory checking](http://disnetdev.com/blog/2012/07/18/memory-checking-in-low-level-javascript/)!
Tim Disney, another intern here at Mozilla, implemented Valgrind style
memory checking for LLJS. It can currently detect the most common
memory errors: use after free, uninitialized reads, double frees, and
memory leaks.

### Esprima in LLJS

As part of another project, I needed super fast JS parsing in
JS. Since I was already working with LLJS, I figured I might as well
try it out on a larger scale and see if I couldn't get
[Esprima](http://esprima.org/) ported over to LLJS and using structs
for the generated AST. I hoped this would make an already fast parser
even faster and leaner by using manual memory allocation.

Well, over 4000 lines of LLJS later, I finished the port. String
handling is somewhat inefficient (it creates a new C string for every
string in the program), but it works! Check out the sources on
[github](http://github.com/rinon/esprima-lljs) if you're interested.

So, was it faster? Well... not really. As it turns out, modern JS
engines are very good at allocating objects, so manual allocation does
not appear to be faster than the engine object allocation. The LLJS
version does use about %10 less memory when parsing large JS sources,
so that's a win. From some preliminary testing, traversing the AST
appears to be faster with LLJS, since property access is fast, but
this comes at the cost of making traversal code harder to write.

Where I think we can definitely win, though, is code which allocates
and frees code very often. Manual memory reuse and freeing could
provide speedups here over engine garbage collection for some
applications (note: I have yet to test this...).


### What's next?

In the beginning of the summer I started off adding source map support
to LLJS, so that debug tools in the browser will show the
corresponding LLJS sources instead of the compiled JS sources. This is
mostly implemented in
[escodegen](http://github.com/Constellation/escodegen/), which we use
to generate the JS code from a rewritten AST. This got a little bogged
down due to a bug in the chrome devtools
([not loading sources from a data URI source-map](http://code.google.com/p/chromium/issues/detail?id=133854)),
so I haven't finished it off yet. Pretty much all that's left to do is
testing there, though.

As far as language features go, I will definitely be adding support
for `enum` types as soon as I get the chance. We are also looking at
implementing [bit fields](https://github.com/mbebenita/LLJS/issues/12)
as a convenience for bit packing in structs.

If you have more ideas for where LLJS should head, please start an
issue on github or (even better) toss us a pull request. Most
importantly though, go try it out! If you find it helpful (or
frustrating), let us know.
