---
layout: post
title: C++ - Non-local Initialization
tags: [cpp]
comments: true
---

Everyone who has been doing C++ long enough knows that
initialization is everything but trivial. You have probably heard of the
most vexing parse, or fought with the 14 different ways of how you can and
about the infamous `std::initializer_list` which killed unicorn
initialization.

Books have been written about C++ initialization, memes have been made.At
this point all we can do is laugh at the ridiculousness of the situation.

I'm here to tell that there is more to it.

# Static Variables

When learning a compiled language we are very soon confronted with the notions of _static_ and _dynamic_.
As the compiler translates your program, it must decide how
to deal with the names introduced: where should the name's
value be put? What should the initial value be, if any? When should it be
destroyed? 

Most of the time your compiler must deal with dynamic
variables, i.e. variables that are initialized at runtime: local
variables, function arguments, non-static class members. Basically all
non-static local names that live and die as part of a scope. Such names
are known as having _automatic storage duration_.

Class members are initialized as an object is instantiated, and their lifetime is bound to the object.

Often when we talk about about variable initialization he are concerned
with variables that have automatic store duration, i.e. local variables.

This makes sense, most variables we have to deal with every day have
automatic storage duration: non-static variables declared at block scope,
function parameters, non-static class members. The lifetime of variables
with automatic storage duration is very clearly defined: their lifetime is
bound to a scope. Local parameters live as long as the corresponding
function frame is in the stack, class members live as long as the
enclosing object lives.

Seems obvious, but what happens to variables that are not defined within
a scope? Global variables most have a lifetime as well, right? They have
to be initialized at some point, and destroyed. They live and die just
like any other, so how does non-local initialization work?
Explain variables with static or thread_local duration

Initialized before main executes (unless static local).

static variable is a general term for variables which location is known at compile time.

# A Word On Global State

Static variables are fine, but global state is problematic. Specially if it is mutable.
You compiler loves it. You functions tell lies, as uncle bob would say.
Here and there you 

# Non-local Initialization

So we have learnt that variables with static storage duration live as long
as the probgram does. But how are they initialized, and most importantly,
when?

Well this should be simple, but off course it isn't. And, as always, you
can do a lot of self damage if you are not careful.

# The Happy Path

Basically things can go two paths: a happy path, and a not so happy one.
When it comes to initialization of non-local variables you must consider
certain things. First

consteval constiniti

# The Danger Zone

constant initialization consteval or constinit

# That's All Folks




