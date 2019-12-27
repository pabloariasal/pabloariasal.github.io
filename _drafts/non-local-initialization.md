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
variables introduced: When should a variable be initialized? where
should its value be put? What's the initial value? When should it be
destroyed?

Most of the time the compiler must deal with _dynamic_ variables, i.e.
variables that are initialized and destroyed at runtime: local
(block-scope) variables, function arguments, non-static class members,
etc.

The compiler has little chance to initialize dynamic variables ahead of
time (e.g. compile time). How is it supposed to know what arguments will
be passed to a function?, or if a given code block will be executed? The answers may even be different from
execution to execution, so their initialization and destruction must
happen on demand, at runtime, _dynamically_.

This is not always the case, though. There is a category of variables
which can be initialized ahead of time: _static
variables_. Global (namespace) variables, static class members, static
local variables have something called _static storage duration_: they live
as long as the program does.

Static variables are initialized before the program starts and destroyed
after the execution finishes. They have the beautiful property that they
can be initialized at compile time.

# Initialization of Static Variables

As discussed, variables with _static storage duration_ live for the entire
execution of the program. They must be initialized once before execution,
and destroyed after. They have the beautiful property that they can be
initialized at compile time.

Initialization of static variables should be simple but off course it
isn't. And, as always, you can do a lot of self damage if you are not
careful.

Initialization of non-local variables with _static storage duration_
happens in two consecutive stages: _static_ and _dynamic_.

Static initialization happens normally at compile time. Initial
values for static variables are evaluated during compilation and burned
into the executable once to be used in all executions. Zero runtime
overhead, early problem diagnosis. This is called
[constant
initialization](https://en.cppreference.com/w/cpp/language/constant_initialization)
and you want it.

After static initialization, dynamic initialization takes place. Dynamic
initialization happens at runtime, before `main` is run[^1]. Here, static
variables are evaluated and initialized when the executable is run and not
during compilation.

In an ideal world, all variables with static storage duration are
initialized at compile time, i.e. they are const-initialized.

## The Green Zone - Constant Initialization

Constant initialization (i.e. compile time) is ideal, that's why your
compiler will try to perform it whenever it can.
This is the case when your variable is initialized by a [constant
expression](https://en.cppreference.com/w/cpp/language/constant_expression),
that is, an something that can be evaluated at compile time.

```cpp
struct MyStruct
{
    static int a;
};
int MyStruct::a = 67; 
```

Here, `MyStruct::a` will be const initialized, because `67` is a compile
time constant, i.e. a constant expression.

### Force Const Initialization with `constexpr`

One way to make sure that variables are const initialized is by
declaring them `constexpr`, this will make it explicit that you want to
use them as constant expressions and force their evaluation and
initialization at compile time.

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

// OK const initialized, enforced by compiler, movePoint must be constexpr
constexpr auto P2 = movePoint({3.f, 4.f}, 1.f, -1.f);

// still const initialized (P2 is constexpr), but not enforced by compiler
const auto P1 = P2;
```

Now consider the following example:

```cpp
struct Line
{
   Line(Point p1, Point p2)
   {
   }
   // ...
};

// error constructor is not constexpr, can't be const initialized
constexpr Line l({6.7f, 5.7f}, {5.4, 3.2});

// OK, but not initialized at compile time
Line l({6.7f, 5.7f}, {5.4, 3.2});
```

`constexpr` must be your first choice when declaring global variables;
very little can go wrong with `constexpr`. `constexpr` variables are not
just initialized at compile time and explicitly forced to be constant
expressions, but `constexpr` implies `const` and immutable state is always
the right way.

### Constinit


constinit does not imply const

## Dynamic Initialization aka gray Zone

Imagine you need an immutable global `std::string` to store the software
version. You probably don't want this object to be instantiated every time
the program runs, but rather have it as initialized and stored into the
executable executable itself. In other words, you want `constexpr`:

```cpp
constexpr auto VERSION = std::string("3.4.1");
```

But life isn't that easy:

```
error: constexpr variable cannot have non-literal type
```

The compiler is complaining because `std::string` doesn't have a trivial
destructor. That means that `std::string` is probably allocating some
resource that must be freed upon destruction, in this case memory. This is
a problem, if we create an `std::string` at compile time the managed
memory must be somehow copied into the executable as well, as it won't be
available when the executable is run!

In other words, `std::string("3.4.1")` is not a constant expression so we
can't force the compiler to const-initialize it.

We must give up:

```cpp
const auto VERSION = std::string("3.4.1");
```

Your compiler is happy, but for the price of moving the initialization
from compile time to runtime. In other words `VERSION` must be initialized
as part of dynamic initialization.

Good news is that at runtime we can allocate memory. Bad news is that this
isn't as nice, safe, or efficient as static initialization, but isn't
inherently bad either. Dynamic initialization will be inevitable in some
cases, not everything can be done or is known at compile time.

The future looks bright, however. Even when moving slowly, smart people
have been working on several proposals to augment the capabilities of
`consexpr` to types like `std::string` or `std::vector`. The idea here is
to allow for memory allocations at compile time and later flash the object
as well as the managed memory into the data section of the executable.
Soon `std::string("3.4.5")` might become constant expression.

# The Static Initialization Fiasco - Red Zone

Things start to get weird if initialization of global variables depend on
each other directly or indirectly


# That's All Folks

* Const initialization happens when a variable is initialized with
  a constant expession


[^1]: Unless the dynamic initialization is deferred, see [here](https://en.cppreference.com/w/cpp/language/initialization)
