---
layout: post
title: Python Descriptors Are Magical Creatures
tags: [python]
related_posts:
  - _posts/2017-11-29-python-decorators-from-the-ground-up.md
---

> It is clear to me that maintaining a popular helper is not worth the time nor the hassle. Without entering into details, this has become an unpaid job I dislike more and more - and I've been talking about it for far too long too. As such, this project is now unmaintained.

Arch Linux forum's user Spyhawk. December 14th, 2017.

The *Arch User Repository*, *AUR* for short, is a community-driven repo that hosts every piece of software ever written by mankind. *AUR* packages are not officially supported by Arch and people have to rely on so-called *AUR* helpers to manage them.

For a long time the land of *AUR* helpers had a single undisputed king: *pacaur*, maintained by my boy Spyhawk. What was once an harmonious monarchy turned into a God-forsaken hell when, in a tragic December night, Spyhawk announced the end of an effort he had driven for over half a decade.

Having been lacking a serious contender for so long, people took *pacaur* for granted and forgot that managing *AUR* packages was a *serious* deal. There was no man brave enough to adopt the 2000 lines of bash left behind by Spyhawk and cheap copycats started appearing all over the place, but the bar was just set too high.

Being stunned by *pacaurs* death and overwhelmed by an endless array of *AUR* helpers I did the only reasonable thing: I wrote another one. That's how *aurepa*, a mixture of *AUR* and arepa, a typical Colombian dish, started to take form. Writing *aurepa* has taken me to explore some of python's darkest corners. Hidden in the shadows I found creatures so smoothly embedded in the python language that you don't even realize they exist. Creatures like python descriptors.

# Getters And Setters Should Be Illegal

Arch Linux packages have two version identifiers: the package version, something like *1.3.5*, and a release number, a monotonically increasing integral that identifies different builds of the same version.
The full version of a package is constructed by concatenating the package version with the package release: *1.3.5-2*.

In my first design of *aurepa* this represented a small inconvenience, I had written a `Package` class that had two attributes: `version` and `release`.

```
>>> p.__class__
<class 'aurepa.Package'>
>>> p.version
'1.3.5'
>>> p.release
2
```

But I also wanted users of `Package` to have access to a `full_version` attribute. Maybe by adding this line to `Package`'s constructor?

~~~python
self.full_version = self.version + '-' + self.release
~~~

This might not be the best idea given that it results in duplicated data: `version` and `release` exist twice. If one of them changes in between `full_version` will become outdated.

You may be having flashbacks of your professor saying something about getter methods. What about this?

~~~python
def get_full_version(self):
    return self.version + '-' + str(self.release)
~~~

No. Hell no. Don't even think about it. This is not Java.

This is inconsistent. This exposes implementation details. When exposing attributes via getters users are forced to make a distinction when accessing them: some are accessed directly with the very nice `object.attribute` notation, others divert to a method invocation. Even when you write a getter to protect access to some attribute, users are not forced to call it, as the protected underlying data is still accessible.

The bottom line is that your object holds a function when it should be holding a data member instead.

Wouldn't it be nice if we could use the `object.attribute` syntax for `full_version` as well?

## The One Word You Need To Remember: Property

Solving our problem is actually horribly simple. All you need is the `@property` decorator:

~~~python
@property
def full_version(self):
    return self.version + '-' + str(self.release)
~~~

That's it! Python's built-in property decorator creates a read-only attribute from a function that computes it, i.e. a getter. Now we can do `p.full_version` just like with any other any other attribute, even if it is calculated dynamically.

```
>>> p.full_version
'1.3.5-2'
```
There is no duplicated data: version and release exist in a single place. There is no inconsistency. Users have transparent access to your attributes.

I could call it a night; my problem is solved. Maybe you should, too... Unless you are feeling adventurous, unless you don't like to take things for granted, unless you posses the same lunatic love for python decorators as I do.

How does `@property` actually work?

# A Journey Into Python's Internals

By using `@property`, python is somehow able to transform a getter function into a data attribute. But how?

First of all let's recall that writing `@property` on top of `func` is equivalent to `func = property(func)`.
But what is `property`? [a class](https://docs.python.org/3.7/howto/descriptor.html). Property is a class and we are invoking its constructor:

~~~python
class property(object):
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)
~~~

So when we write:

~~~python
@property
def full_version(self):
~~~

We are creating an object of class property by passing the function `full_version` as the first parameter to its constructor. Remember that functions in python can be passed around like any other object, because they are, well, objects.
~~~python
full_version = property(full_version)
~~~

So we replaced `full_version` in the `Package` class by an instance of `property`. This means that when users access `package.full_version` they should get back an object of type `property`, right?

```
>>> p.full_version
'1.3.5-2'
>>> type(p.full_version)
<class 'str'>
```

When `full_version` is accessed the getter method we decorated is somehow being executed behind the scenes. `full_version` doesn't seem to be of type property after all, even thought we just replaced it by one! The *magic* behind this lies in the definition of the methods `__get__`, `__set__`, and `__del__` in the `property` class. These methods implement an interface, better said a *protocol*, python's data descriptor protocol.

# Python's Descriptor Protocol

Studying the implementation of `property` created more questions than answers.
We have found out that decorating a getter method with `@property` replaces it by an instance of the property class. This new object stores the decorated getter function, `self.fget = fget`, and executes it inside `__get__()`: `return relf.fget(obj)`.

However, when we access the property object we don't get the object itself, but rather the result of its `__get__` method. But why is `__get__` being executed? The answer is simple: magic. That's just how it is, if an object declares a method called `__get__`, python won't return the object itself when accessed, but the result of its `__get__`.

Objects that define either of these methods:

~~~python
__get__(self, obj, type=None)
__set__(self, obj, value)
__del__(self, obj)
~~~

Are called *descriptors*. If an objects implements all three, like `property`, it's called a *data descritor*. If only `__get__` is present, the object is called a *non-data descriptor*.

Descriptors can define custom behavior when they are being accessed, modified or deleted. In other words, they *override* python's default attribute access. When the interpreter encounters `o.x` it looks up the `__dict__` of `o` for an entry named *x*. If `x` defines a `__get__`, then `o.x` is translated to `type(o).__dict__['x'].__get__(o, type(o))`. If not, the default `o.__dict__['x']` is returned.

A similar *magic* happens when the attribute is being modified, e.g. `o.x = 5`, or deleted `del x`. `__set__` and `__del__` will be called, respectively, instead of performing default behavior.

Note that data descriptors have higher precedence than instance variables in `__dict__`. If you define a class that contains a data descriptor, e.g. a property, they will opaque all other instance variables with the same name, since `a.property = 'some_new_value'` will execute `a.property.__set__` instead of replacing it. Hence, data descriptors can't be overridden in objects. Non-data descriptors, on the other hand, don't define a `__set__` and can therefore be reassigned in the object's `__dict__`.

I discovered that a lot of python's functionality relies in the descriptor protocol: `@staticmethod`, `@classmethod` are implemented as non-data descriptors. Even normal method invocation is!

## Property Objects Are Data Descriptors

The implementation of `property` now becomes clear: it replaces the decorated function by a data descriptor. When the attribute is accessed `__get__` is called which in turn calls `fget`, the decorated function. Properties are just wrappers around our function that implement the data descriptor protocol.

If you pay close attention you will notice that `property` also implements `__delete__` and `__set__`. This is important because the definition of these methods makes property objects data descriptors. Key in this implementation is that both `__delete__` and `__set__` raise an `AttributeError` per default when executed. This is great! I don't want users to alter the version of my package or delete it, it should be a read-only attribute! Turns out you can actually achieve proper data encapsulation in python after all.


# Writable Data Attributes

It is sometimes useful to have modifiable properties that define custom behavior when users call `del obj.property` or `obj.property = some_new-value`. You can achieve this by passing the corresponding setter and deleter functions as arguments when constructing the `property` object. The new object will store them and execute them when the attribute is written or deleted, i.e. from `__set__` and `__delete__`, respectively.

~~~python
class Computer():
	def __init__(self):
		self._operating_system = None

	def get_operating_system(self):
		return self._operating_system

	def set_operating_system(self, os):
		if 'Linux' not in os:
			raise AttributeError('You should use Linux you fool!')
		self._operating_system = os
		print('{} has been installed!'.format(os))

	operating_system = property(get_operating_system, set_operating_system)
~~~

```
>>> laptop = Computer()
>>> laptop.operating_system = 'Windows 10'
...
AttributeError: You should use Linux you fool!
>>> laptop.operating_system = 'Arch Linux'
Arch Linux has been installed!
```

This works but it is a rather rustic solution that forces use to use a class attribute. Wouldn't it be nice if we could use decorators? Unfortunately, you can't create a modifiable attribute by using `@property` alone, as per definition only the first argument will be passed when instantiating property.

The class `Property`, however, defines convenience functions for setters and deleters as well, so that you can use them as decorators:

~~~python
class Property():

	def getter(self, fget):
		return type(self)(fget, self.fset, self.fdel, self.__doc__)

	def setter(self, fset):
		return type(self)(self.fget, fset, self.fdel, self.__doc__)

	def deleter(self, fdel):
		return type(self)(self.fget, self.fset, fdel, self.__doc__)
~~~

This let's you do the following:

~~~python
class Computer():
	def __init__(self):
		self._operating_system = None

	@property
	def operating_system(self):
	    return self._operating_system

	@operating_system.setter
	def operating_system(self, os):
		if 'Linux' not in os:
			raise AttributeError('You should use Linux you fool!')
		self._operating_system = os
		print('{} has been installed!'.format(os))
~~~

This took me a while to understand but once you see it's fairly simple. The functions `setter`, `getter` and `deleter` are methods of the property class but also decorators. But what do they decorate? They decorate *self*, i.e. the property object itself they belong to!

The setter decorator, for example, receives the decorated `fset` as argument and constructs a new property object by passing it to its constructor. Note, however, that existing functions in `self` (`fget` or `del`) are also propagated to the new object. This is exactly what decorators are all about, they receive some object, add some stuff to it (in this case `fset`) and return it.

*aurepa* might not be the successor of *pacaur* and I might not become the man that Spyhawk once was, but at least this journey has given me, and you, a glimpse of an obscure python feature I didn't know existed but use every day. Maybe that cold December night wasn't that tragic after all, like some ying and yang kind of thing.
