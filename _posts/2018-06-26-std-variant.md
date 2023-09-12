---
layout: post
title: std::variant Doesn't Let Me Sleep
tags: [cpp]
---

There is a saying in Spanish that I like very much: *de la risa al llanto*, which translates to *from laughter to tears*. It's used in situations that appear beneficial at first, but end up turning into a curse; when the line between the bad and the good dissipates, when the dream becomes the nightmare.

Last month I attended a talk by Juanpe Bolivar: [the most valuable values](https://www.youtube.com/watch?v=NMol_5-2owo). In his talk he made use of C++17's new sum type proposal: `std::variant`. Sum types are objects that can vary their type dynamically.
A `std::variant` is similar to a union: it allocates a fixed portion of memory and reuses it to hold a value of one of several predefined alternative types at a time:

~~~cpp
#include <variant>

std::variant<int, float, char> a = 42; //a is an int
a = 3.141f;                            //a holds now a float
char my_char = std::get<char>(a);      //error: a holds a float, not a char
a = 'c';                               //all cool bro
~~~

`std::variant`s are superior to unions because they are type-safe: the variant object knows which type is currently being held. Self harm becomes a little bit more difficult if you use variants over unions.

Often you want to do something different with a variant depending on the type currently being held. In such cases you need to *visit* the variant with `std::visit`. `std::visit` receives two parameters: the variant itself and a visitor, which is a function object that is callable for all types supported by the variant:

~~~cpp
struct Visitor
{
    void operator()(int a)
    {
        //Called if variant holds an int
    }
    void operator()(float a)
    {
        //Called if variant holds a float
    }
    void operator()(char a)
    {
        //Called if variant holds a char
    }
};

std::visit(a, Visitor{});
~~~

In his talk Juanpe presented a convenience function for creating such visitors, `make_visitor()`, that constructs a visitor like the one above from a list of lambdas:

~~~cpp
auto visitor = make_visitor(
    [](int b)
    {
        //Called if variant holds an int
    },
    [](float b)
    {
        //Called if variant holds a float
    },
    [](char b)
    {
        //Called if variant holds a char
    }
);
~~~

`make_visitor()` is not part of the standard and it instantly caught my eye. How would the implementation of such function look like? I closed my eyes and after some consideration I managed to scribble something like this in my head:

~~~cpp
template<typename ...Ts>
struct Visitor : Ts...
{
    Visitor(const Ts&... args) : Ts(args)...
    {
    }
}

template<typename ...Ts>
auto make_visitor(Ts... lamdbas)
{
    return Visitor<Ts...>(lambdas...);
}
~~~

This isn't precisely first semester programming class, but I assure you that it looks much more scary than it really is. For the above code to make any sense there are two language features that need to be understood: parameter packs and closure classes. Let's revisit them very quickly.

# Parameter Packs

I usually tell people that my favorite C++11 feature are parameter packs, also known in the streets as variadic templates. A parameter pack is a template that can be instantiated with any number of types. For example, you can use parameter packs to define functions that take arbitrary arguments:

~~~cpp
template<typename ...Ts>
void f(Ts... args)
{
}
~~~

`f()`, `f(8, 9.0f`), `f("Hello")` are all possible calls to `f`. For every call, the compiler will perform template type deduction and instantiate definitions of `f` with `Ts = {}`, `Ts = {int, float}` and `Ts={const char*}`, respectively.

You may have noticed the ellipsis operator `...` in `f(Ts... args)`. This is called a *pack expansion* and it lets you unroll a parameter pack. Unrolling basically means constructing a comma-separated list from a pattern containing a parameter pack.

For example, if `Ts={int, char}`, `Ts...` will expand the pack to `int, char`.
If you give the pack a name, the name will be expanded as well: `Ts... args` becomes `int args0, char args1`.
This also works on more complex patterns: `const Ts&... args`, will be `const int& args0, const char& args1`, `Ts(args)...` expands to `int(args0), char(args1)`. I hope you get the idea.

The *Visitor* struct above inherits from all types in its parameter pack `Ts`. This is achieved by expanding `Ts` in the inheritance list: `struct Visitor: Ts...`. Note that pack expansion is also used to define a copy constructor that propagates the call to the base classes constructors.

This is how an instantiation of `Visitor<A,B,C>` would look like after expansion:

~~~cpp
struct Visitor : A, B, C
{
    Visitor(const A& arg0, const B& arg1, const C& arg2) : A(arg0), B(arg1), C(arg2)
    {
    }
}

Visitor<A,B,C> make_visitor(A lambda1, B lambda2, C lambda3)
{
    return Visitor<A,B,C>(lambda1, lambda2, lambda3);
}
~~~

# Closure Classes

The implementation of `make_visitor(Ts... args)` above returns an object that inherits from all the types in `Ts`. If this function is called with a set of lambdas, which is what we want, this object will be a struct inheriting from all lambdas provided. Wait, what? Can you inherit from a lambda? What is actually the type of a lambda?

Yes, you can inherit from a lambda and the type of a lambda is, well, only your compiler knows.
Despite their incredible impact after their introduction, lambdas are just syntactic sugar. Lambdas are just a shortened notation for constructing callable, a.k.a function objects.

Consider the following example:

~~~cpp
const int n{42};
auto lambda = [n](int a){return n + a;};
const auto m = lambda(8); //m is now 50
~~~

When the compiler sees this it will translate `lambda` into a function object. This object will be instantiated from a compiler-generated class specifically for this lambda, called a closure class.
This is the the generated closure class for `lambda`:

~~~cpp
class ClosureClass
{
public:
    ClosureClass(int i): n(i)
    {
    }

    int operator()(int a)
    {
        return n + a;
    }

private:
    int n;
}
~~~

Note that the captured variables of the lambda are translated as member variables in the closure class.

The snippet above effectively becomes:

~~~cpp
const int n{42};
auto lambda = ClosureClass(n);
const auto m = lambda(8); //m is now 50
~~~

Hopefully by now the implementation of `make_visitor()` should be clear: it returns an object that inherits from all closure classes deduced from the lambdas provided as argument. Each one of the closure classes defines an `operator()` for a different type in the variant, creating an overload set in the derived class, just like the very first visitor we built.

You might be amazed by my intelligence and creativity, but don't get fooled. This doesn't work.

# From Laughter to Tears

Some nights ago while browsing reddit I stumbled across a [blog post](https://bitbashing.io/std-visit.html) by Matt Klein on, guess what, `std::visit` . In his post, Matt makes a reasonable critique on the growing complexity of C++ and how absurdly difficult it sometimes is to solve common tasks. To defend his thesis Matt presented the same problem as Juanpe: construct a visitor for a variant from a set of lambdas. This time, however, he provided an implementation:

~~~cpp
template <class... Ts>
struct Visitor;

template <class T, class... Ts>
struct Visitor<T, Ts...> : T, Visitor<Ts...>
{
    Visitor(T t, Ts... rest) : T(t), Visitor<Ts...>(rest...) {}

    using T::operator();
    using Visitor<Ts...>::operator();
};

template <class T>
struct Visitor<T> : T
{
    Visitor(T t) : T(t) {}

    using T::operator();
};

~~~

I studied Matt's solution for some minutes but was unable to fully grasp his intentions. I closed my browser and went to sleep, hoping to forget about it in the morning. I didn't. Nights went by and I became more anxious; I couldn't sleep. Why does Matt need the recursive overload? Why are those `using operator()` all over the place? Matt saw something I didn't and I wasn't going to rest until I had seen it too.

But what is Matt doing anyway?

# Recursive Variadic Templates

As we saw early, parameter packs let you to define templated classes and functions that can be instantiated with arbitrary types:

~~~cpp
template<typename ...Ts>
void f(Ts... args)
{
}
~~~

But what if you want to do anything with the parameters of `f()` at all? Like, for example, write all arguments of `f()` to stdout?
In that case you need to define a recursive overload set:

~~~cpp
template<typename T>
void f(const T& t)
{
    std::cout << t << std::endl;
}

template<typename T, typename ...Ts>
void f(const T& head, const Ts&... tail)
{
    std::cout << head << ", ";
    f(tail...);
}
~~~

Recursive overload sets are a common pattern when dealing with variadic templates. The key idea is simple: split the parameter pack into two parts: a head and a tail. The head is the first parameter in the pack, the tail are the rest.

The pattern is always the same. You provide two overloads for your function: a recursive case and a base case.
The recursive variant does something with the head and recursively call itself with the tail. Note that the size of the pack decreases after each recursive call of `f()`.

`f()` will keep invoking itself with a smaller list each time until the base case is reached: the parameter pack contains just the last item. Here the compiler will, during overload resolution, call the overload that receives a single argument: `f(const T& t)`. This breaks the recursion and we are done.

Matt does uses the same strategy in his implementation: he defines an overloaded recursive struct. In the recursive case, his struct inherits from head and recursively from an instantiation of itself using tail. Yes, you heard that right, Matt's visitor inherits from itself:

~~~cpp
template <class T, class... Ts>
struct Visitor<T, Ts...> : T, Visitor<Ts...>
{
    Visitor(T t, Ts... rest) : T(t), Visitor<Ts...>(rest...) {}

    using T::operator();
    using Visitor<Ts...>::operator();
};
~~~

The recursion continues to unroll constructing an inheritance hierarchy with all types in the parameter pack, until the base case is reached and the recursion stops.

Matt brings the `operator()` of the super classes (remember we are dealing with closure classes here) into the scope of the derived class. He does this recursively: `operator()`s from all classes in the hierarchy end in the scope of the very bottom class.

# Reaching Nirvana

Similarly to my original idea, Matt constructs an object that inherits from all the closure classes in the pack. But what kept me awake at night was the usage of the recursive overload and the pulling of the `operator()` into the derived classes scope.
Luckily a miraculous search landed me on the [C++ FAQ](https://isocpp.org/wiki/faq/strange-inheritance#overload-derived):

> In C++, there is no overloading across scopes – derived class scopes are not an exception to this general rule.

Overload sets don't work across multiple scopes! Off course, `A::operator()` and `B::operator()` inherited from two base classes don't overload each other in the derived class. I wasn't building an overload set in *Visitor* at all! And then it finally hit me: Matt needs the recursive overload because he needs to bring all `operator()` into the scope of the bottom class to form an overload set!

I don't know is Matt's solution is a strike of genious or the product of a very troubled mind.

This story still has a happy ending, though. My solution was not entirely wrong, it was, well, ahead of its time. The C++17 standard introduces pack expansion with the `using` keyword:

~~~cpp
template<typename ...Ts>
struct Visitor : Ts...
{
    Visitor(const Ts&... args) : Ts(args)...
    {
    }

    using Ts::operator()...;
}
~~~

I sleep like a baby ever since.
