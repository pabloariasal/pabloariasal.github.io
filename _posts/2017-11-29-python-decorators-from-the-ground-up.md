---
layout: post
title: Python Decorators From the Ground Up
tags: [python]
permalink: python-decorators-from-the-ground-up/
related_posts:
  - _posts/2018-11-25-python-descriptors.md
---

Decorators are your friends. Despite this they tend to be seen as a rather obscure feature and are often left horribly neglected.
In this post I will show you that decorators don't need to be complicated and that, when used correctly, can become your MVP in many situations.

I won't just tell you what decorators are, but rather build a bottom up journey where we assemble a decorator from scratch, step by step. I'll visit the different ideas that make up the decorator idiom and the specific issues they aim to solve. Hopefully by the end of your reading you will be able to identify the scenarios where decorators excel and the reason they became part of the python language.

Let's begin with a very realistic example.

# The Algorithm of Love

Imagine you are working on a dating platform that lets users find hot singles in their area.
Your matching algorithm, based on some vague search criteria, is able to retrieve perfect matches from the user registry:

~~~python
import time

def hottie_lookup(search_criteria):
    print('Querying hotties database...')
    search_results = []
    #Simulate patended algorithm to find hot singles in your area
    time.sleep(2)
    return search_results
~~~

In no time your platform becomes very popular. Even though people really seem love it, you have heard some complaints about long response times.
You decide to measure the execution time of your love algorithm:

~~~python
import time

def hottie_lookup(search_criteria):

    current_time = time.time()

    print('Querying hotties database...')
    search_results = []
    time.sleep(2)
    #Simulate patended algorithm to find hot singles in your area

    print("Request took {:.1f}s".format(time.time()-current_time))

    return search_results
~~~

After the great success you decide that it would be useful to gather some usage statistics:
What are the most popular searching criteria (besides hotness off course)? What are the most popular singles in town?

For this you decide to log every received request and its corresponding response:

~~~python
import time

def hottie_lookup(search_criteria):

    print("Request was made:", search_criteria)
    current_time = time.time()

    print('Querying hotties database...')
    search_results = []
    #Simulate patented algorithm to find hot singles in your area
    time.sleep(2)

    print("Request took {:.1f}s".format(time.time()-current_time))
    print("Matching entries found:", search_results)

    return search_results
~~~

Your matching algorithm works so well that the majority of your users found their true love, and, as your analytics show, a clear decay in traffic is evident.
You meditate on this and discover that your algorithm could be generalized to solve other kinds of matching problems, such as finding the more skilful gardeners in town:

~~~python
import time

def gardener_lookup(search_criteria):

    print("Request was made", search_criteria)
    current_time = time.time()

    print('Querying gardeners database...')
    search_results = []
    #Simulate patended algorithm to find skillful gardeners in town
    time.sleep(2)

    print("Request took {:.1f}s".format(time.time()-current_time))
    print("Matching entries found:", search_results)

    return search_results
~~~

Perfect! You can now use your magical matching solution for the sake of perfectly mowed lawns and less awkward family gatherings.
Now, is this the best way of solving this? not really, there are a few issues with this design:

### Code Duplication

It's clear that in our services, hotties and gardeners search, functionality has to be replicated: both need to be timed and logged.
This is bad, since every time we need to add a new service the same stuff needs to be reimplemented.

Image that we receive the requirement to authenticate users before each database lookup, then we would then need to implement user authenticaion in every single service.
What if we had a few dozen services? Embarrassing.

### Single Responsibility Principle

In our functions, which are supposed to run awesome ranking algorithms to retrieve perfect matches from the database, we are suddenly measuring time and persisting analytics.
Our functions are polluted, have several reasons to change and break the single responsibility principle. Shame.

# Wrapper Functions

Clearly polluting our lookup functions with extra functionality isn't the best idea.
But, how can we time our functions while leaving them unchanged? Maybe we can _wrap_ them inside another function:

~~~python
import time

def hottie_lookup_timed(search_criteria):
    current_time = time.time()
    # Call original unchanged hottie_lookup
    search_results = hottie_lookup(search_criteria)
    print("Request took {:.1f}s".format(time.time()-current_time))
    return search_results
~~~

Is this better? well, kinda, at least `hottie_lookup()` doesn't need to worry about measuring execution time anymore.
But what about request and response logging? we could write a wrapper function of our wrapper function dawg:

~~~python
def hotties_lookup_logged(search_criteria):
    print("Request was made:", search_criteria)
    search_results = hottie_lookup_timed(search_criteria)
    print("Matching entries found:", search_results)
    return search_results
~~~

Awesome! Now `hottie_lookup()` is back to its original implementation and free of extra work.

All we need to do now is search/replace all calls to `hottie_lookup()` with `hotties_lookup_logged()`, so that every time the service is executed the wrapper function is called instead.
Sure, but if we want to add an extra wrapper? Then we would need to rename all function calls again :/.
Sure we can do better.

# Functions Are First Class Citizens

By enclosing our lookup functions inside a wrapper, we were able to perform timing and logging by leaving the original implementations unchanged. This is a step in the right direction, but there are still some issues to solve.

First off all, we completely forgot about the gardener search, that function needs to be decorated as well!
Sure we can repeat the same trick, but we would end up having two functions, `gardener_lookup_timed()` and a `gardener_lookup_logged()` that would be practically the same their hottie search versions.
What if we want to log the time in milliseconds instead of seconds? We would need to change two implementations.

What do we do in this cases? We generalize! Python is a very democratic language, where functions have the same rights as any other value, so why not just write a single timer wrapper function that receives the function to be timed as parameter?

~~~python
import time

def timer(func):
    def inner_func(search_criteria):
        current_time = time.time()
        result = func(search_criteria)
        print("Request took {:.1f}s".format(time.time()-current_time))
        return result
    return inner_func
~~~

Wow wow slow down, what happened here? Yes, it looks complex, but it really isn't that complicated once we have dissected it.

Decorators take advantage of how functions are implemented in python.
Have you ever heard of classes and objects? well in python functions are just that, objects.
This means that functions can be passed as arguments to another functions and returned as any other value.
Once you have understood this the snippet above becomes much more digestible.

A decorator is  a function that receives a function and returns a function.
Think about it this way: we are **decorating**, we are supposed to receive something, add some extra stuff to it and return the same thing we received.
But, what exactly is returned? The wrapper function from last section! All decorators do is create a wrapper function and return it.

This new "trick" is so awesome that we can write our `hottie_lookup_timed()` and `gardener_lookup_timed()` like this:

~~~python
hottie_lookup_timed = timer(hottie_lookup)
gardener_lookup_timed = timer(gardener_lookup)
~~~

Do you see it? our timer decorator takes our original function, adds a timer around it, and returns it.
And most importantly: timing functionality exists in a single place and can be used to time _any_ function we want. Do we have to log time in milliseconds? Sure, a single modification to our timer decorator will do.

Let's make a test with our new `hotties_lookup_timed()` function with some very popular search criteria:

~~~python
>>> hotties_lookup_timed(['latino', 'computer scientist'])
Querying hotties database...
Request took 2.0 seconds
~~~

Perfect! Our function is timed and the decorator seems to be working.

Naturally the same technique can be applied to construct a single logging decorator.

## Generalizing with _*args_ and _**kwargs_

The timer decorator we wrote works, but only if we are decorating a function that receives a single argument.
What if in the future we need to decorate a function that receives no arguments? That wouldn't work with the current implementation, as our inner function declares one argument.

The problem here is that we are making assumptions on the function to be decorated.

For this reason, in case your decorators don't need to know anything about the decorated function, it is standard practice to define our inner function with variable arguments and propagate them to the input function:

~~~python
import time
def timer(func):
    def inner_func(*args, **kwargs):
        current_time = time.time()
        result = func(*args, **kwargs)
        print("Request took {:.1f}s".format(time.time()-current_time))
        return result
    return inner_func
~~~

Don't let python's _*args_ and _**kwargs_ scare you away. All we did was define a function that can receive an arbitrary number of arguments, both keyworded and non-keyworded. For example, you can do this: `inner_func(8)`, or this: `inner_func(city='Munich', state='Bavaria')`. You then propagate all your arguments to `func`, that's it.

By using _*args_ and _**kwargs_, our decorators can decorate arbitrary functions as we no longer need to know beforehand the arguments they declare.

# Leaving Callers Unchanged

We were able to decorate our `hotties_lookup()` and `gardeners_lookup()` with a shared timer decorator.
But are we completely happy? not really, we still have the problem mentioned before: callers of our functions need to be changed, as `hotties_lookup_timed()` needs to be called instead of `hotties_lookup()`.
But remember, functions are objects, we can just assign them to other variables, so why not just reassign the original implementations?

~~~python
hotties_lookup = timer(hotties_lookup)
gardeners_lookup = timer(gardeners_lookup)
~~~

We just _replaced_ our original `hotties_lookup()` with its decorated version.
This means that callers no longer need to be changed, since `hotties_lookup()` now points to the timed (a.k.a decorated) version:

~~~python
>>> hotties_lookup(['latino', 'computer scientist'])
Querying hotties database...
Request took 2.0 seconds
~~~

This idiom is so common that it has its own built-in language syntax.

The expression:

~~~python
hotties_lookup = timer(hotties_lookup)
~~~
is equivalent to:

~~~python
@timer
def hotties_lookup(search_criteria):
    pass
~~~

# Putting It All Together

Now we are happy. Here the entire implementation:

First we write the decorators we need:

~~~python
import time

def logger(func):
    def inner_func(search_criteria):
        print("Request has made with criteria:", search_criteria)
        result = func(search_criteria)
        print("Matching entries found:", result)
        return result
    return inner_func

def timer(func):
    def inner_func(search_criteria):
        current_time = time.time()
        result = func(search_criteria)
        print("Request took {:.1f}s".format(time.time()-current_time))
        return result
    return inner_func
~~~

Then we decorate our functions:

~~~python
@logger
@timer
def hottie_lookup(search_criteria):
    print('Querying hotties database...')
    search_results = []
    #Simulate patended algorithm to find hottes singles in your area
    time.sleep(2)
    return search_results

@logger
@timer
def gardener_lookup(search_criteria):
    print('Querying gardeners database...')
    search_results = []
    #Simulate patended algorithm to find skilful gardeners in town
    time.sleep(2)
    return search_results
~~~

~~~python
>>> hotties_lookup(['latino', 'computer scientist'])
Request has made with criteria: ['latino', 'computer scientist']
Querying hotties database...
Request took 2.0s
Matching entries found: []

>>> gardeners_lookup(['bush sculpture', 'semilegal botany'])
Request has made with criteria: ['bush sculpture', 'semilegal botany']
Querying gardeners database...
Request took 2.0s
Matching entries found: []
~~~

Voila. That's it. We have covered the basics of python decorators and hopefully you are now convinced of how simple and useful they are. This is not the end of it, though, there is more stuff you can do with decorators. Most importantly, you can add extra generalization to your decorators by passing arguments to them:

~~~python
@time('seconds')
def hottie_lookup(search_criteria):
    pass

@time('milliseconds')
def gardener_lookup(search_criteria):
    pass
~~~

Here, for instance, we wrote a generic timer decorator that receives the time unit to measure as argument. This lets us time the hottie lookup in seconds and the gardener lookup in milliseconds, all by using a single decorator.
Decorators with arguments are out of the scope of this post, but if you are curious and have liked this post, leave me  a comment and maybe I'll write a follow up post.

# Wrapping Up

* Decorators receive a function and returned a wrapped version of it
* Decorated functions or methods don't change: decorators can only append or prepend functionality
* Decoration is transparent to callers: original function calls are replaced by decorated versions
* Decorators avoid code duplication: decorating functionality resides in a single place
* Decorators are reusable: decorate several functions with single decorator
* Decorators are flexible: combine decorators as you wish, you can even decorate decorators!
* Use _*args_ and _**kwargs_ to write generic decorators that can decorate arbitrary functions
