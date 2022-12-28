---
layout: post
title: C++ - Initialization of Static Variables
tags: [cpp]
---

Today is the last day of the year. I'm wearing my yellow underwear; it's a new year's tradition in this part of the world. People say it shall bring wealth, luck and happiness for the upcoming twelve months. Growing up, I used to consider it silly to wear yellow underwear on new year's eve. Today I think silly is the one who doesn't.

You are probably reading this because you code in C++. This means that you have battled frustration mastering `auto` deduction rules or lost your sanity trying to understand why `std::initializer_list` was considered a good idea. Anyone who has been doing this long enough knows that variable initialization is everything but trivial. It's a problem too essential to ignore but too challenging to master. Today I'm here to tell you that there is more to it.

# Static Variables

As the compiler translates your program it must decide how to deal with
variables introduced: When should a variable be initialized? What's the
initial value? When should it be destroyed?

Most of the time the compiler must deal with _dynamic_ variables, i.e.
variables that are initialized and destroyed at runtime: local
(block-scope) variables, function arguments, non-static class members,
etc.

The compiler has little chance to initialize such variables before
execution starts: How is it supposed to know what arguments will be passed
to a function?, or if a given code block will be reached? The answers may
even vary from execution to execution, so their initialization and
destruction must happen on demand, at runtime, _dynamically_.

There is, however, a category of variables that can (and should) be
initialized before the program starts: static variables. Global
(namespace) variables or static class members[^2] live for the entire
execution of the program: they must be initialized before `main()` is
run and destroyed after execution finishes. Such variables have
_static storage duration_.

The lifetime of static variables doesn't depend on the
execution: they always exist; forever; no matter what.
This leads to the beautiful property that they
can be potentially evaluated and initialized at compile time.

# The Two Stages of Static Variable Initialization

As discussed, variables with _static storage duration_ must be initialized
once before the program starts and destroyed after execution terminates.
Initialization of static variables should be simple, but off course it
isn't. And, as always, you can do a lot of self damage if you are not
careful.

Initialization of static variables happens in two consecutive stages:
_static_ and _dynamic_ initialization.

Static initialization happens first and usually at compile time. If possible, initial
values for static variables are evaluated during compilation and
burned into the data section of the executable. Zero
runtime overhead, early problem diagnosis, and, as we will see later, safe. This
is called [constant
initialization](https://en.cppreference.com/w/cpp/language/constant_initialization).
In an ideal world all static variables are const-initialized.

If the initial value of a static variable can't be evaluated at compile time, the compiler will perform zero-initialization.
Hence, during static initialization all static variables are either const-initialized or zero-initialized.

After static initialization, dynamic initialization takes place. Dynamic
initialization happens at runtime for variables that can't be evaluated at
compile time[^1]. Here, static variables are initialized every
time the executable is run and not just once during compilation.

# The Green Zone - Constant Initialization

Constant initialization (i.e. compile time) is ideal, that's why your
compiler will try to perform it whenever it can.
This is the case when your variable is initialized by a [constant
expression](https://en.cppreference.com/w/cpp/language/constant_expression),
that is, an expression that can be evaluated at compile time.

```cpp
//a.cpp
struct MyStruct
{
    static int a;
};
int MyStruct::a = 67;
```

Here, `MyStruct::a` will be const-initialized, because `67` is a compile
time constant, i.e. a constant expression[^3].

## Force Const Initialization with `constexpr`

One big problem with static variable initialization is that it is not
always clear if a variable is being initialized at compile time or at
runtime.

One way to make sure that variables are const-initialized (i.e. compile time) is by declaring
them `constexpr`, this will force the compiler to treat them as constant expressions and perform their evaluation and initialization at compile time.

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

// OK const-initialized, enforced by compiler
constexpr auto P2 = movePoint({3.f, 4.f}, 1.f, -1.f);

// still const-initialized, but not enforced by compiler
auto P1 = P2;

struct Line
{
   Line(Point p1, Point p2)
   {
   }
   // ...
};

// Compile error, l can't be const-initialized (no constexpr constructor)
constexpr Line l({6.7f, 5.7f}, {5.4, 3.2});
```

`constexpr` must be your first choice when declaring global variables (assuming you really need a global state to begin with).
`constexpr` variables are not just initialized at compile time, but `constexpr` implies
`const` and immutable state is always the right way.

## Your Second Line of Defense: `constinit`

`constinit` is a keyword introduced in the c++20 standard. It works just
as `constexpr`, as it forces the compiler to evaluate a variable at compile
time, but with the difference that it __doesn't imply const__.

As a consequence, variables declared `constinit` are always const-(or
zero-)initialized, but can be mutated at runtime, i.e. don't land in
read-only data section in the binary.

```cpp
constexpr auto N1{54}; // OK const-init ensured
constinit auto N2{67}; // OK const-init ensured, mutable

int square(int n)
{
  return n * n;
}
constinit auto N3 = square(N2); // Compilation error square(N2) not constant expression

int main()
{
  ++N2; // OK
  ++N1; // Compilation error
  return EXIT_SUCCESS;
}

```

# The Yellow Zone - Dynamic Initialization

Imagine you need an immutable global `std::string` to store the software
version. You probably don't want this object to be instantiated every time
the program runs, but rather create it once and embed it into the
executable as read-only memory. In other words, you want
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
cases, not everything can be done at compile time, for
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

# The Red Zone - Static Initialization Order Fiasco

Dynamic initialization of static variables suffers from a very scary
defect: the order in which variables are initialized at runtime is not
always well-defined.

Within a single compilation unit, static variables are initialized in the
same order as they are defined in the source (this is called *Ordered Dynamic Initialization*). Across compilation units,
however, the order is undefined: you don't know if a static variable
defined in `a.cpp` will be initialized before or after one in `b.cpp`.

This turns into a very serious issue if the initialization of a variable
in `a.cpp` depends on another one defined `b.cpp` . This is called
the [Static Initialization Order Fiasco](https://isocpp.org/wiki/faq/ctors#static-init-order).
Consider this example:

```cpp
// a.cpp
int duplicate(int n)
{
    return n * 2;
}
auto A = duplicate(7); // A is dynamic-initialized
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

This program is ill-formed. It may print `14` or `0` (all static variables
are at least zero-initialized during static initialization),
depending if the dynamic initialization of `A` happens before `B` or not.

Note that this problem can only happen during the dynamic initialization
phase and not during static initialization, as during compile time it is
impossible to access a value defined in another compilation unit. This
makes compile time initialization so much safer than dynamic
initialization, as it doesn't suffer from the static initialization order
fiasco.

## Solving The Static Initialization Order Fiasco

Encountering the static initialization order fiasco is often a symptom of poor software design. IMHO the best way to solve it is by refactoring the code to break the initialization dependency of globals across compilation units. Make your modules self-contained and strive for constant initialization.

If refactoring is not an option, one common solution is the [Initialization On First Use](https://isocpp.org/wiki/faq/ctors#static-init-order-on-first-use). The basic idea is to design your static variables that are not constant expressions (i.e. those that must be initialized at runtime) in a way that they are created *when they are accessed for the first time*. This approach uses a static local variable inspired by the [Meyer's Singleton](http://laristra.github.io/flecsi/src/developer-guide/patterns/meyers_singleton.html).
With this strategy it is possible to control the time when static variables are initialized at runtime, avoiding use-before-init.

```cpp
// a.cpp
int duplicate(int n)
{
    return n * 2;
}

auto& A()
{
  static auto a = duplicate(7); // Initiliazed first time A() is called
  return a;
}
```

```cpp
// b.cpp
#include <iostream>
#include "a.h"

auto B = A();

int main()
{
  std::cout << B << std::endl;
  return EXIT_SUCCESS;
}

```

This program will always consistently print `14`, as it is guaranteed that `A` will always be initialized before `B`.

# That's All Folks

* Static variables must be initialized before the program starts
* Variables that can be evaluated at compile time (those initialized by a constant expression) are const-initialized
* All other static variables are zero-initialized during static initialization
* `constexpr` forces the evaluation of a variable as a constant expression and implies const
* `constinit` forces the evaluation of a variable as a constant expression and doesn't imply const
* After static initialization dynamic initialization takes places, which happens at runtime before `main()`
* Within a compilation unit static variables are initialized in the order of declaration
* The order of initialization of static variables is undefined across compilation units
* You can use the Initialization On First Use Idiom to circumvent the static initialization oder fiasco

[^1]: Unless the dynamic initialization is deferred, [see](https://en.cppreference.com/w/cpp/language/initialization)
[^2]: [see](https://en.cppreference.com/w/cpp/language/storage_duration)
[^3]: The compiler can promote initialization for non-constant expressions to compile time under certain conditions, [see](https://en.cppreference.com/w/cpp/language/initialization)
