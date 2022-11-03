---
layout: post
title: Playing with io_uring and C++20 Coroutines - Part 1
tags: [cpp]
comments: true
---

As part of my job I have to deal with very heavy file I/O. The problem is always the same: read hundreds of files into memory as efficiently as possible.

This is a problem that every systems programmer will face sooner or later. There are many different ways of approaching this problem, for me it was the trigger to finally play around with `io_uring`, the new Linux API for high-performance asynchronous I/O. With the C++20 standard including coroutines into the language, and compiler having matured enough in their implementation, it became an irresistible hobby project. Asynchronous I/O and coroutines go as well together as bread and butter.

In this post we'll write a program that reads and parses hundreds of OBJ files stored on disk using Linux `io_uring` and C++20 coroutines. Its not meant to be an in-depth guide of either one of these topics (even though the basics are explained), nor be the most efficient implementation possible, but rather to showcase the fantastic synergy that they form when combined.

The motivation comes from the fact that there are lots of in-depth resources for both `io_uring` and C++20 coroutines, but practically nothing showing how to write a program that uses both.

The series will be split into two parts. In the first part we will lay the ground work and solve the problem using `io_uring` only, in the second part we will re-write the program with C++20 coroutines.

# Motivation

Asynchronous I/O, as opposed to synchronous I/O, is based on the idea of performing an I/O operation without blocking the calling thread. In other words, the thread performing the operation is released immediately, and can do other work as the requested operation is performed in the background. After a while the thread collects the results of the operation. Its like when you tell your pizza guy: "Giovanni, put a margarita in the oven, I'll go grab something from the pharmacy and will be right back".

Asynchronous I/O can be a lot more efficient than synchronous I/O, threads don't need to wait for resources to become available, but at the same time it makes programs more complex. Programs need to work, well, asynchronously, they must remember to pick up the margaritas they ordered. Coroutines allow us to write asynchronous code as if it was synchronous. If you write a coroutine that loads a file from disk, it can suspend execution and return to the caller while the file is being loaded, and resume once the data is available. All written as if it was a good-old synchronous call.

Combining both worlds allows us to write asynchronous programs without actually writing asynchronous code. The best of both worlds.

# Goals

In this post series we will be writing a program that reads and parses hundreds of OBJ files from disk.

Its important to understand the distinction between reading and parsing. Reading denotes the action of loading the contents of a file from disk into a memory buffer. Parsing means taking the data in the buffer and transforming into into a data structure that makes sense to the application.

An OBJ is an ASCII file that describes the geometry of one or more 3D shapes, represented as triangular meshes. The file encodes information like the triangles that make up the shape, their colors, surface normals, etc. Parsing an OBJ file means converting its ASCII representation into a C++ objects that has a convenient API.

// image of obj

For parsing the OBJ files we will use the great [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader) library. It receives a string containing the contents of the OBJ file and parses it into a `ObjReader` object:

```cpp
tinyobj::ObjReader reader;
reader.ParseFromString(obj_data);
```

We can then use the `reader`s to access the attributes of the shapes defined in the file, to list how many shapes are defined in the file you can do the following:

```cpp
std::cout << reader.GetShapes().size() << '\n';
```

The goal of our program is to load a list of OBJ files and print the number of shapes contained in each one of them.

# Starting out: Ground work

First of all let's lay down some ground work. We will define some basic types and abstractions that will make our implementation easier and safer, mainly by wrapping C APIs within RAII objects.

We will be dealing with files and buffers, so let's introduce abstractions for both:

```cpp
```
Quite simple, opens a file in read only mode and closes it on destruction.

Now let's implement a Buffer type:

```cpp

```

We will be allocating a buffer per file. This very inefficient. There many ways of reducing the allocation overhead, one can for example make use of arenas (see "region-based memory management") or some other sort of buffer recycling strategy. For simplicity purposes we will stick to one buffer per file, as this makes the implementation substantially simpler and its irrelevant for the purpose of the blog post.

Another type we will use a lot is the result:

```cpp

```
This is the result of the parsing of an OBJ file, it contains the final OBJ object, the corresponding file path, as well as the status code of the read call.

# A Trivial Approach

Now that we have defined some ground types, we can start with the main dish.

As usual, let's start with the easiest possible implementation: a single-threaded blocking file loader.

```cpp
// readSynchronous
```

`readSynchronous` receives a file, loads its contents into a buffer and then parses the data in the buffer into an `OBJ` object. `readObjFromBuffer` wraps around `tinyobjloader` and initializes the `result` member of `Result`.

All we need to do now is call `readSynchronous` for every file and print its result.
```cpp

// void trivial and error handling
```

This is easy, but slooow. The `read` syscall blocks the calling thread until the data has been read into the buffer. And I mean *the thread*, there is only one thread that waits for I/O **and** does the parsing for all files. We can't start processing the next file until the parsing of the previous one finished!

The context switches between user and kernel mode are not harmless either, every time that we call `read` we switch into kernel mode. If we load hundreds of files we will have hundreds of context switches between user and kernel space.

We can do better.

# Better: A Multi-Threaded Implementation

I know what you are thinking, just parallelize!

```cpp
// void thread_pool
```

Here we are using the amazing [thread-pool library](https://github.com/bshoshany/thread-pool) (please go on github and give it some love) to run individual iterations of the for-loop in parallel. Every thread receives a fixed number of iterations, denoted by the range [a,b). You can also use an openMP pragma or any other implementation, the idea is the same.

This is better, even when the threads are still being blocked on the `read` syscall, we are now processing multiple files in parallel. The overhead in terms of code changes is very small, and it opens further optimization opportunities: we can for example allocate a single buffer per thread that can be reused by multiple files.

For many applications this approach is more than sufficient, but imagine that you developing a web server, listening to hundreds of socket connections at once. Would you create a thread per socket? Probably not. What you need is a way of telling the OS: "I'm interested in these sockets, let me know when any of them has any data to be read". What you need is asynchronous I/O.

# Linux `io_uring`
`io_uring` is a new asynchronous I/O API available in the Linux kernel (version 5.1 upwards). This is not the first API of its kind, there are other options available in the kernel, like `epoll`, `poll`, `select` and `aio`, all with their issues and limitations. `io_uring` aims to start a new page and become the standard API for all asynchronous I/O operations in the kernel.

The API is called `io_uring` because its based on two ring buffers: the submission queue and the completion queue. The buffers are shared between user code and the kernel and writing to or reading from them does not require system calls or copies. Ring buffers are an efficient way for implementing FIFO queues.

The idea is simple: user code writes requests into the submission queue and submits them to the kernel, the kernel consumes the requests in the queue, performs the requested operations, and writes the results into the completion queue. User code can asynchronously retrieve the completed requests from the queue at a later point.

![gollum]({{ site.baseurl }}/assets/img/gollum_iouring.png "gollum")

The raw API of `io_uring` is a little complex, this is why programs usually use a wrapper library called `liburing` (created by Jens Axboe, original author of `io_uring`), which extracts most of the boilerplate away and provides convenient utilities for using `io_uring`.

We will now implement our program using `liburing`.

First we define a thin wrapper RAII class that creates the `io_uring` and frees it upon destruction.

```cpp
```

First of all we must ask the kernel to load the files. For this we create a `io_uring` instance, initialize the buffers, and write entries into the submission queue, one per file.

```cpp

// first part of the function

```

`io_uring_get_sqe` creates an entry in the submission queue, `sqe`. We can now set the contents of the submission entry with `io_uring_prep_read`, with specifies that we want the kernel to read the contents of the file pointed to by `files[i].fd()` into the buffer `buffs[i]`.

One can also append user data to the submission entry with `io_uring_sqe_set_data`. This is data that is not used by the kernel but is just copied into the corresponding completion entry of the current request. This is important in order to differentiate which completion entry corresponds to which submission entry. In this case we just write the index of the file, which we can use to reference back when the read operation completes.

After we have written all the submission entries into the queue its time to submit them to the kernel and wait until they start appearing in the completion queue. Once completion entries appear, we can read the corresponding buffers into OBJ files.

```cpp
```

First we call `io_uring_submit_and_wait`, which submits all entries to the kernel and blocks until at least one completion request has been pushed into the completion queue.

Once we have completion entries in the queue, we can start processing them. `io_uring_for_each_cqe` is a macro whose body gets executed for every completion entry in the completion queue.

This is what we must do for every completion entry:

1. get index of the file this completion entry corresponds to.
This is the same id we wrote into the corresponding submission entry.
2. Write the status code of read call into the `Result` and check it. This is the return code of the `read` operation.
3. Parse the OBJ from the buffer into the `result` field `Result`.

Finally, we can liberate some space in the submission queue as we have processed some entries. For this we use `io_uring_cq_advance`, which does nothing more than moving head of the ring buffer back by n elements, making room for more entries.

The first big advantage of this implementation is that we greatly reduced the number of system calls performed. In the synchronous implementations we made a `read` syscall for every file, now we make a single syscall for all the requests in the queue. Reading and writing to the submission and completion queues happens in user space, without switching into kernel mode.

This approach is definitely better than the trivial approach, but still has some issues. First, the code is substantially more complex than the synchronous case. And second, our program is CPU-bound! As some profiling shows, most of the time is spent parsing the OBJs:

```
  %   cumulative   self
 time   seconds   seconds
 31.25      0.05     0.05    tinyobj::tryParseDouble(char const*, char const*, double*)
 18.75      0.08     0.03    tinyobj::LoadObj(tinyobj::attrib_t*, std::vector<tinyobj::shape_t, std::allocator<tinyobj::shape_t> >*, std::vector<tinyobj::material_t, std::allocator<tinyobj::material_t> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::istream*, tinyobj::MaterialReader*, bool, bool)
 12.50      0.10     0.02    allDone(std::vector<Task, std::allocator<Task> > const&)
 12.50      0.12     0.02    tinyobj::parseReal(char const**, double)
  6.25      0.13     0.01    tinyobj::parseReal3(float*, float*, float*, char const**, double, double, double)
  ...
  ```

The parsing of the OBJ files still happens on a single thread!
While processing the submission entries happens in parallel inside the kernel, consuming the completion entries happens in a single thread in user space. We must parallelize the parsing path.

In the next part of the series we will try to solve these problems. First, we will rewrite the current implementation by using C++20 coroutines.
To give you a sneak peek, we will implement something that looks similar to a function:

```cpp
Result loadFile(const ReadOnlyFile &file, IOUring& uring) {
  // ...
}
```
but its not **not** a function, its a coroutine! This function will add a read file request to the submission queue but will _immediately suspend execution_, returning to the caller. The coroutine will be waken up and resume execution once the corresponding completion entry arrives into the submission queue.

Afterwards we will improve the implementation so that the coroutines are waken up on a worker thread, conducting the parsing of OBJs in parallel. This is easily achievable with coroutines.

Interested? [Read part 2]()

# Part 2

In this second part of the series we will rewrite our program by making use of C++ coroutines.

The idea is simple: we want to implement a coroutine that loads and parses an OBJ file from disk asynchronously with `io_uring`. When this coroutine is called it should push a submission request into the queue and suspend execution, returning to the caller. The coroutine can be resumed later once the completion entry has arrived and the parsing takes place.

# A Coroutine Implementation

## What do we want?

```cpp
// loadfile

```
## ReadFileAwaitable
add both the pushing and polling part (only blocking)

## Task
## coroutines function

# Multithreaded
// loadFilePool
// ThreadPool

TODO:

strace

We want to co_await a read operation, when we are woken up we want to start in a different thread.

In the last part we wrote a program that reads and parses hundreds of OBJ files using `io_uring`. We will now rewrite the program by making use C++20 coroutines.

# C++20 Coroutines

A coroutine is a function that can suspend execution. Suspending execution means to store the current execution context of the "function" (local values, parameters, current execution point) and return control back to the caller, as would happen after a normal `return`. The difference is that after beign suspended the coroutine remains alive, dormant, but alive and can be resumed at a later point.

When a coroutine is suspended all local variables, parameters and resume point are stored in a heap-allocated object called the coroutine frame. The coroutine is controlled with a so-called coroutine handle, which can be used to resume the coroutine or destroy the coroutine frame.

A coroutine can be resumed at a later point in time, here the execution context (i.e local variables) is restored from the coroutine frame and execution resumes right from where the coroutine was last suspended, until either the coroutine ends or the next suspension point is encountered.

In C++ coroutines are stackless, this means that even thought the behave similar to functions there is no stack frame being created. The execution of the coroutine happens "magically".

A suspension point of a coroutine can be denoted with the keywords `co_await`, `co_yield` or `co_return`. `co_yield` suspends execution of the coroutine by yielding a value, similar to what python's `yield`.

```cpp
generator<int> count_until(int n) {
  for (int i = 0; i < n; ++i)
  {
    co_yield ++i;
  }
}
```

Which can be used like:

```cpp
auto generator = count_until(5);
while(!generator.isDone()) {
  std::cout << generator() << '\n';
}
```
will print:
```
1
2
3
4
5
```

`generator` is a type that we have to write ourselves, the *coroutine type*. It internally stores the coroutine handle which is used to resume to coroutine. `generator` overloads `operator()` from where it calls `this->coroutine_handle().resume()`. This causes the coroutine to resume execution until the next `co_yield` is encountered, yielding the next value and returning again control back to the caller.

Once we have called `generator()` enough times the coroutine will finish. We can use the coroutine handle to check if a coroutine has finished running by calling `this->coroutine_handle.done()`. This is what we do in the `isDone()` member function of `generator`.

The lifetime of the coroutine frame must be managed by us, this is why the desctructor of `generator` must call `this->coroutine_handle.destroy()`, which will deallocate the coroutine frame.

`co_return` is similar to `co_yield` with the difference that it completes execution of the coroutine.

## Coroutine Promise

C++20 coroutines are raw, this means that instead of being a finished cake, they are more like a bunch of flour, eggs and butter. In order to implement a coroutine you have to write a lot of supporting code and boilerplate, you must bake your own cake. For this reason it is usual to make use of a coroutine library like [cppcoro by Lewis Baker](https://github.com/lewissbaker/cppcoro), which implements above's `generator` type, among a lot of more useful things. BTW go check out Lewis Baker [blog series](https://lewissbaker.github.io/) on coroutines if you haven't so. Truly marvelous.

One object that we have o write when implementing a coroutine is the *promise object*.

// promise object in the context of co_yield

# Awaitables

More interesting for us is `co_await`. `co_await` is an operator that expects an *awaitable*, or something that can be converted into an awaitable via `operator co_await()`. `co_await` suspends the coroutine. An awaitable is a type that
