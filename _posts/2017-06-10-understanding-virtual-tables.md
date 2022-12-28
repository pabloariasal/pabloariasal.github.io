---
layout: post
title: Understandig Virtual Tables in C++
tags: [cpp]
---

> "A virtual method table (VMT),..., is a mechanism used in a programming language to support dynamic dispatch." --[Wikipedia](https://en.wikipedia.org/wiki/Virtual_method_table)
{:.lead}

## What does _dynamic dispatch_ mean?

In this context, dispatching just refers to the action of finding the right function to call. In the general case, when you define a method inside a class, the compiler will remember its definition and execute it every time a call to that method is encountered.

Consider the following example:

~~~cpp
#include <iostream>

class A
{
public:
  void foo();
};

void A::foo()
{
  std::cout << "Hello this is foo" << std::endl;
}
~~~
Here, the compiler will create a routine for `foo()` and remember its address. This routine will be executed every time the compiler finds a call to `foo()` on an instance of `A`. Keep in mind that only one routine exists per class method, and is shared by all instances of the class. This process is known as _static dispatch_ or _early binding_: the compiler knows which routine to execute during compilation.

### So, what do _vtables_ have to do with all this?

Well, there are cases where it is not possible for the compiler to know which routine to execute at compile time. This is the case, for instance,  when we declare virtual functions:

~~~cpp
#include <iostream>

class B
{
public:
  virtual void bar();
  virtual void qux();
};

void B::bar()
{
  std::cout << "This is B's implementation of bar" << std::endl;
}

void B::qux()
{
  std::cout << "This is B's implementation of qux" << std::endl;
}
~~~

The thing about virtual functions is that they can be overriden by subclasses:

~~~cpp
class C : public B
{
public:
  void bar() override;
};

void C::bar()
{
  std::cout << "This is C's implementation of bar" << std::endl;
}
~~~

Now consider the following call to `bar()`:
~~~cpp
B* b = new C();
b->bar();
~~~

If we use static dispatch as above, the call `b->bar()` would execute `B::bar()`, since (from the point of view of the compiler) b points to an object of type `B`. This would be horribly wrong, off course, because b actually points to an object of type `C` and `C::bar()` should be called instead.

Hopefully you can see the problem by now: given that virtual functions can be redefined in subclasses, calls via pointers (or references) to a base type can not be dispatched at compile time. The compiler has to find the right function definition (i.e. the most specific one) at runtime. This process is called _dynamic dispatch_ or _late method binding_.

## So, how do we implement dynamic dispatch?

For every class that contains virtual functions, the compiler constructs a _virtual table_, a.k.a _vtable_.
The _vtable_ contains an entry for each virtual function accessible by the class and stores a pointer to its definition. Only the most specific function definition callable by the class is stored in the _vtable_. Entries in the _vtable_ can point to either functions declared in the class itself (e.g. `C::bar()`), or virtual functions inherited from a base class (e.g. `C::qux()`).

In our example, the compiler will create the following virtual tables:

![vtables](/assets/img/posts/vtables/vtables.png "vtables")

The vtable of class `B` has two entries, one for each of the two virtual functions declared in `B`'s scope: `bar()` and `qux()`. Additionally, the _vtable_ of `B` points to the local definition of functions, since they are the most specific (and only) from `B`'s point of view.

More interesting is `C`'s _vtable_. In this case, the entry for `bar()` points to own `C`'s implementation, given that it is more specific than `B::bar()`. Since `C` doesn't override `qux()`, its entry in the _vtable_ points to `B`'s definition (the most specific definition).

Note that _vtables_ exist at the class level, meaning there exists a single _vtable_ per class, and is shared by all instances.

### Vpointers

You might be thinking: _vtables_ are cool and all, but how exactly do they solve the problem?
When the compiler sees `b->bar()` in the example above, it will lookup `B`'s _vtable_ for `bar`'s entry and follow the corresponding function pointer, right? We would still be calling `B::bar()` and not `C::bar()`...

Very true, I still need to tell the second part of the story: _vpointers_. Every time the compiler creates a _vtable_ for a class, it adds an extra argument to it: a pointer to the corresponding virtual table, called the _vpointer_.

![vpointer](/assets/img/posts/vtables/vpointer.png "vpointer")

Note that the _vpointer_ is just another class member added by the compiler and increases the size of every object that has a _vtable_ by `sizeof(vpointer)`.

Hopefully you have grasped how dynamic function dispatch can be implemented by using _vtables_: when a call to a virtual function on an object is performed, the _vpointer_ of the object is used to find the corresponding _vtable_ of the class. Next, the function name is used as index to the _vtable_ to find the correct (most specific) routine to be executed. Done!

## Virtual Destructors

By now it should also be clear why it is always a good idea to make destructors of base classes virtual. Since derived classes are often handled via base class references, declaring a non-virtual destructor will be dispatched statically, obfuscating the destructor of the derived class:

~~~cpp
#include <iostream>

class Base
{
public:
  ~Base()
  {
    std::cout << "Destroying base" << std::endl;
  }
};

class Derived : public Base
{
public:
  Derived(int number)
  {
    some_resource_ = new int(number);
  }

  ~Derived()
  {
    std::cout << "Destroying derived" << std::endl;
    delete some_resource_;
  }

private:
  int* some_resource_;
};

int main()
{
  Base* p = new Derived(5);
  delete p;
}
~~~

This will output:

```
> Destroying base
```

Making `Base`'s destructor virtual will result in the expected behavior:

```
> Destroying derived
> Destroying base
```

## Wrapping up

* Function overriding makes it impossible to dispatch virtual functions statically (at compile time)
* Dispatching of virtual functions needs to happen at runtime
* The virtual table method is a popular implementation of dynamic dispatch
* For every class that defines or inherits virtual functions the compiler creates a virtual table
* The virtual table stores a pointer to the most specific definition of each virtual function
* For every class that has a _vtable_, the compiler adds an extra member to the class: the _vpointer_
* The _vpointer_ points to the corresponding vtable of the class
* Always declare desctructors of base classes as virtual
