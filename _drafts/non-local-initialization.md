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

# Static Variables

As the compiler translates your program it must decide how to deal with
variables introduced: When should a variable be initialized? where should
its value be put? What's the initial value? When should it be destroyed?

Most of the time the compiler must deal with _dynamic_ variables, i.e.
variables that are initialized and destroyed at runtime: local
(block-scope) variables, function arguments, non-static class members,
etc.

The compiler has little chance to initialize dynamic variables before
execution starts: How is it supposed to know
what arguments will be passed to a function?, or if a given code block
will be executed? The answers may even vary from execution to execution,
so their initialization and destruction must happen on demand, at runtime,
_dynamically_. Such variables have _automatic storage duration_.

There is, however, a category of variables that can (and should) be
initialized before the program starts: static variables. Global
(namespace) variables or static class members[^2] live for the entire
execution of the program: they must be initialized before `main()` is run
and destroyed after execution finishes. Such variables have _static
storage duration_.

Note that the lifetime of variables with static storage duration doesn't vary
from execution to execution: they always exist; forever. This leads to the beautiful
property that they can be initialized and evaluated at compile time.

# Initialization of Static Variables

As discussed, variables with _static storage duration_ must be initialized
once before the program starts and destroyed after execution terminates.
Initialization of static variables should be simple, but off course it
isn't. And, as always, you can do a lot of self damage if you are not
careful.

Initialization of static variables happens in two consecutive stages:
_static_ and _dynamic_.

Static initialization happens first and usually at compile time. Initial
values for static variables are evaluated during compilation once and
burned into the executable for prosperity. Zero runtime overhead, early
problem diagnosis, and, as we will see later, safe. This is called
[constant
initialization](https://en.cppreference.com/w/cpp/language/constant_initialization).
In an ideal world, all static variables are const-initialized.

After static initialization, dynamic initialization takes place. Dynamic
initialization happens at runtime, before `main()` is run[^1]. Here,
static variables are evaluated and initialized every time the executable
is run and not once during compilation.

## The Green Zone - Constant Initialization

Constant initialization (i.e. compile time) is ideal, that's why your
compiler will try to perform it whenever it can.
This is the case when your variable is initialized by a [constant
expression](https://en.cppreference.com/w/cpp/language/constant_expression),
that is, an expression that can be evaluated at compile time.

```cpp
struct MyStruct
{
    static int a;
};
int MyStruct::a = 67; 
```

Here, `MyStruct::a` will be const-initialized, because `67` is a compile
time constant, i.e. a constant expression.

### Force Const Initialization with `constexpr`

One big problem with static variable initialization is that it is not
always clear what is happening behind the scenes, are they being
initialized at runtime or at compile time? 

One way to make sure that variables are const-initialized is by declaring
them `constexpr`, this will force their evaluation and initialization at
compile time and make it explicit that you want to intend to use them as
constant expressions.

```cpp
struct Point
{
    float x{};
    float y{};
};

constexpr Point movePoint(Point p, float x_offset, float y_offset)
{
    return {p.x + x_offset, p.y + y_offset};
}

struct Line
{
   // not constexpr
   Line(Point p1, Point p2)
   {
   }
   // ...
};

// OK const-initialized, enforced by compiler, movePoint must be constexpr
constexpr auto P2 = movePoint({3.f, 4.f}, 1.f, -1.f);

// still const-initialized (P2 is a constant expression), but not enforced by compiler
const auto P1 = P2;

// Error l can't be const-initialized (no constexpr constructor)
constexpr Line l({6.7f, 5.7f}, {5.4, 3.2});
```

`constexpr` must be your first choice when declaring global variables.
`constexpr` variables are not just initialized at compile time and
explicitly forced to be constant expressions, but `constexpr` implies
`const` and immutable state is always the right way.

### Your Third Line of Defense: `constinit`

`constinit` is a keyword introduced in the c++20 standard. It works just
as `constexpr`, as it forces the compiler to initialize a variable at compile
time, but with the difference that it _doesn't imply const_.

As a consequence, variables declared `constinit` are always const-(or
zero-)initialized but can be mutated at runtime, i.e. don't land in
read-only data section in the binary.

```cpp
constexpr auto N1{54}; // OK const-init ensured
constinit auto N2{67}; // OK const-init ensured, mutable

int square(int n)
{
  return n * n;
}
constinit auto N3 = square(N2); // Error square() not constant expression

int main()
{
  ++N2; // OK
  ++N1; // Error
  return EXIT_SUCCESS;
}

```

`constinit` should be your third resort when declaring global variables.
I say third because your first choice should be avoid any sort of global
state altogether. Your second option is `constexpr`; if you are to have
a global state, at least make it immutable. A very common case for `constinit` is a mutex // TODO


## The Yellow Zone - Dynamic Initialization

Imagine you need an immutable global `std::string` to store the software
version. You probably don't want this object to be instantiated every time
the program runs, but rather create it once and embed it into the
executable itself as read-only memory. In other words, you want
`constexpr`:

```cpp
constexpr auto VERSION = std::string("3.4.1");
```

But life isn't that easy:

```
error: constexpr variable cannot have non-literal type
```

The compiler is complaining because `std::string` defines a non-trivial
destructor. That means that `std::string` is probably allocating some
resource that must be freed upon destruction, in this case memory. This is
a problem, if we create an `std::string` at compile time the managed
memory must be somehow copied into the binary as well, as it won't be
available when the executable is run!

In other words, `std::string("3.4.1")` is not a constant expression so we
can't force the compiler to const-initialize it!

We must give up:

```cpp
const auto VERSION = std::string("3.4.1");
```

Your compiler is happy, but for the price of moving the initialization
from compile time to runtime. `VERSION` must be initialized as part of
dynamic initialization, not static.

Good news is that at runtime we can allocate memory. Bad news is that this
isn't as nice, safe, or efficient as static initialization, but isn't
inherently bad either. Dynamic initialization will be inevitable in some
cases, not everything can be done at compile time, or is known, for
example:

```cpp

// generate a random number every time executable is run
const auto RANDOM = generateRandomNumber();

```

The future looks bright, however. Even when moving slowly, smart people
have been working on several proposals to augment the capabilities of
`constexpr` to types like `std::string` or `std::vector`. The idea here is
to allow for memory allocations at compile time and later flash the object
alongside its managed memory into the data section of the binary.

# The Static Initialization Fiasco - Red Zone

Dynamic initialization of static variables suffers from a very scary
symptom: the order of initialization is not always
well-defined.

Within a single compilation unit, static variables are initialized in the
same order as they are defined in the source code. Across compilation
units, however, the order is undefined. This turns into a very serious
issue if the initialization of a variable depends on the value of another
one defined in a different compilation unit. This is called the [Static
Initialization Order
Fiasco](https://isocpp.org/wiki/faq/ctors#static-init-order):

```cpp 
// a.cpp
int duplicate(int n)
{
    return n * 2;
}
auto A = duplicate(7); // dynamic initialized
```

```cpp 
// b.cpp
#include <iostream>

extern int A;
auto B = A; // dynamic initialized

int main()
{
  std::cout << B << std::endl;
  return EXIT_SUCCESS;
}

```

This program is ill-formed. It may print `14` or it may print `0` (all
static variables are at least zero-initialized at compile time), depending
on if `A` is initialized before `B` or not. Undefined behavior in its
purest state.

Note that this problem can only happen during the dynamic initialization
phase, as during compile time it is impossible to access
a value defined in another compilation unit. This makes compile time
initialization so much safer, as it doesn't suffer from the static
initialization fiasco.

# That's All Folks

* Const initialization happens when a variable is initialized with
  a constant expession


[^1]: Unless the dynamic initialization is deferred, see [here](https://en.cppreference.com/w/cpp/language/initialization)
[^2]: [see](https://en.cppreference.com/w/cpp/language/storage_duration)
[^3]: The compiler may delay the initialization of a static variable to when it's actually used[here](https://en.cppreference.com/w/cpp/language/initialization)
