---
layout: post
title: C++20 Coroutines and io_uring - Part 1
tags: [cpp]
comments: true
---

As part of my job I have to deal with very heavy file I/O. The problem is always the same: read hundreds of files into memory as efficiently as possible.

This is a problem that every systems programmer will face sooner or later. There are many different ways of approaching it, for me it was the trigger to play around with `io_uring`, the new Linux API for high-performance asynchronous I/O. With coroutines being accepted into the C++20 standard and compilers having matured enough in their implementations, it became an irresistible hobby project.

In this post we'll write a program that reads and parses hundreds of [wavefront OBJ](https://en.wikipedia.org/wiki/Wavefront_.obj_file) files stored on disk using Linux `io_uring` and C++20 coroutines. Its not meant to be an in-depth guide of either one of these topics, nor be the most efficient implementation possible, but rather to showcase the synergy that forms when used together. Asynchronous I/O and coroutines go as well together as bread and butter.

The motivation comes from the fact that there are lots of in-depth resources for both `io_uring` and C++20 coroutines, but practically nothing showing how to combine both.

<img src="{{ site.baseurl }}/assets/img/ohne_rahmen.png" width="500" height="auto">

The series will be split into three parts. In the first part we will lay the ground work and solve the problem using `io_uring` only, in the second part we will re-write the program on top of C++20 coroutines. In the third part we will extend the coroutines implementation by including multi-threading.

# Motivation

Asynchronous I/O, as opposed to synchronous I/O, is based on the idea of performing an I/O operation without blocking the calling thread. In other words, the thread performing the operation is released immediately, and can do other work as the requested operation is performed in the background. After a while the thread collects the results of the operation. Its like when you tell your pizza guy: "Giovanni, put a margarita in the oven, I'll go grab something from the pharmacy and will be right back".

Asynchronous I/O can be a lot more efficient than synchronous I/O, threads don't need to wait for resources to become available, but at the same time it makes programs more complex. Programs need to work, well, asynchronously, they must remember to pick up the margaritas they ordered. Coroutines allow us to write asynchronous code as if it was synchronous. If you write a coroutine that loads a file from disk, it can suspend execution and return to the caller while the file is being loaded, and resume once the data is available. All written as if it was a good-old synchronous call.

Combining asynchronous I/O and coroutines allows us to write asynchronous programs without actually writing asynchronous code. The best of both worlds.

# Goals

In this post series we will be writing a program that reads and parses hundreds of OBJ files from disk.

Its important to understand the distinction between reading and parsing. Reading denotes the action of loading the contents of a file from disk into a memory buffer. Parsing means taking the data in the buffer and transforming into into a data structure that makes sense to the application.

An OBJ is an ASCII file that describes the geometry of one or more 3D triangular meshes. The file encodes information like the triangles that make up the mesh, their vertices, colors, surface normals, etc. Parsing an OBJ file means converting its ASCII representation into a C++ objects that has a convenient API.

<img src="{{ site.baseurl }}/assets/img/Dolphin_triangle_mesh.png" width="350" height="auto">
[source](https://en.wikipedia.org/wiki/Triangle_mesh)

For parsing the OBJ files we will use the great [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader) library. It receives a string containing the contents of the OBJ file and parses it into a `ObjReader` object:

```cpp
tinyobj::ObjReader reader;
reader.ParseFromString(obj_data);
```

We can use the API of `reader` to access the attributes of the shapes defined in the file. For example, to list how many shapes are defined in the file you can do the following:

```cpp
std::cout << reader.GetShapes().size() << '\n';
```

# Ground Work

First of all let's lay down some ground work. We will define some basic types and abstractions that will make our implementation easier and safer, mainly by wrapping C APIs within RAII objects.

We will be dealing with files and buffers, so let's introduce abstractions for both:

```cpp
class ReadOnlyFile {
public:
  ReadOnlyFile(const std::string &file_path) : path_{file_path} {
    fd_ = open(file_path.c_str(), O_RDONLY);
    if (fd_ < 0) {
      throw std::runtime_error("Could not open file: " + file_path +
                               ". Error: " + strerror(errno));
    }
    size_ = get_file_size(fd_);
    if (size_ < 0) {
      throw std::runtime_error("Could not estimate size of file");
    }
  }

  ReadOnlyFile(ReadOnlyFile &&other)
      : path_{std::exchange(other.path_, {})},
        fd_{std::exchange(other.fd_, -1)},
        size_{other.size()} {}

  ~ReadOnlyFile() {
    if (fd_) {
      close(fd_);
    }
  }

  int fd() const { return fd_; }
  off_t size() const { return get_file_size(fd_); }
  const std::string &path() const { return path_; }

private:
  std::string path_;
  int fd_;
  off_t size_;
};
```
Quite simple, opens a file in read only mode and closes it on destruction.
We do some basic error handling and implement a move constructor. This is required to store `ReadOnlyFile` inside some STL containers like `std::vector`.

Analogous is the implementation of a buffer type:

```cpp
class Buffer {
public:
  explicit Buffer(off_t size) : size_{size} {
    buff_ = malloc(size);
    if (!buff_) {
      throw std::runtime_error("could not initialize file");
    }
  }

  Buffer(Buffer &&other)
      : buff_{std::exchange(other.buff_, nullptr)},
      size_{other.size_}
      {}

  ~Buffer() {
    if (buff_) {
      free(buff_);
    }
  }

  void *get() const { return buff_; }
  off_t size() const { return size_; }

private:
  void *buff_{nullptr};
  off_t size_;
};
```

We will be allocating a buffer per file. This very inefficient. There many ways of reducing the allocation overhead, one can for example make use of arenas (see [region-based memory management](https://en.wikipedia.org/wiki/Region-based_memory_management)) or some other sort of buffer recycling strategy. For simplicity purposes we will stick to one buffer per file, as its irrelevant for the purpose of the post.

Another type we will use a lot is `Result`:

```cpp
struct Result {
  tinyobj::ObjReader result; // stores the actual parsed obj
  int status_code{0};        // the status code of the read operation
  std::string file;          // the file the OBJ was loaded from
};
```
This is the final result of the parsing of an OBJ file, it contains the final OBJ object, the corresponding file path, as well as the status code of the read call.

The goal of our program is to load a list of OBJ files and return a `std::vector<Result>`.

# A Trivial Approach

Now that we have defined some ground types we can continue with the main dish.

As usual, let's start with the easiest possible implementation: a single-threaded blocking file loader.

```cpp
Result readSynchronous(const ReadOnlyFile &file) {
  Result result{.file = file.path()};
  Buffer buff{file.size()};
  read(file.fd(), buff.get(), buff.size()); // blocks current thread
  readObjFromBuffer(buff, result.result);
  return result;
}
```

`readSynchronous` receives a file, loads its contents into a buffer, and then parses the data in the buffer into an `OBJ` object. `readObjFromBuffer` wraps around tinyobjloader and initializes the `result` member of `Result`.

```cpp
void readObjFromBuffer(const Buffer &buff, tinyobj::ObjReader &reader) {
  auto s = std::string((char *)buff.get(), buff.size());
  reader.ParseFromString(s, std::string{});
}
```

All we need to do now is call `readSynchronous` for every file:

```cpp
std::vector<Result> trivialApproach(const std::vector<ReadOnlyFile> &files) {
  std::vector<Result> results;
  results.reserve(files.size());
  for (const auto &file : files) {
    results.push_back(readSynchronous(file));
  }
  return results;
}
```

This is easy, but slooow. The `read` syscall blocks the calling thread until the data has been read into the buffer. And I mean *the thread*, there is only one thread that waits for I/O **and** does the parsing for all files. We can't start processing the next file until the parsing of the previous one finished!

The context switches between user and kernel mode are not harmless either, every time that we call `read` we switch into kernel mode. If we load hundreds of files we will have hundreds of context switches between user and kernel space.

We can do better.

# Next Attempt: Use a thread pool

I know what you are thinking, just parallelize!

```cpp
std::vector<Result> threadPool(const std::vector<ReadOnlyFile> &files) {
  std::vector<Result> result(files.size());
  BS::thread_pool pool;
  pool.parallelize_loop(files.size(),
                        [&files, &result](int a, int b) {
                          for (int i = a; i < b; ++i) {
                            result[i] = readSynchronous(files[i]);
                          }
                        })
      .wait();
  return result;
}
```

Here we are using the [thread-pool library](https://github.com/bshoshany/thread-pool) (please go on github and give it some love) to run individual iterations of the for-loop in parallel. Every thread receives a fixed number of iterations, denoted by the range [a,b). You can also use openMP, the idea is the same.

This is better, even when the threads are still being blocked on the `read` syscall, we are now processing multiple files in parallel. The overhead in terms of code changes is very small, and it opens further optimization opportunities: we can for example allocate a single buffer per thread that can be reused by multiple files.

For many applications this approach is more than sufficient, but imagine that you developing a web server, listening to hundreds of socket connections at once. Would you create a thread per socket? Probably not. What you need is a way of telling the OS: "I'm interested in these sockets, let me know when any of them has any data to be read". What you need is asynchronous I/O.

# Linux `io_uring`

`io_uring` is a new asynchronous I/O API available in the Linux kernel (version 5.1 upwards). This is not the first API of its kind, there are other options available in the kernel, like `epoll`, `poll`, `select` and `aio`, all with their issues and limitations. `io_uring` aims to start a new page and become the standard API for all asynchronous I/O operations in the kernel.

The API is called `io_uring` because its based on two ring buffers: the submission queue and the completion queue. The buffers are shared between user code and the kernel and writing to or reading from them does not require system calls or copies. Ring buffers are an efficient way for implementing FIFO queues.

The idea is simple: user code writes requests into the submission queue and submits them to the kernel, the kernel consumes the requests in the queue, performs the requested operations, and writes the results into the completion queue. User code can asynchronously retrieve the completed requests from the queue at a later point.

<img src="{{ site.baseurl }}/assets/img/gollum_iouring.png" width="500" height="auto">

The raw API of `io_uring` is a little complex, this is why programs usually use a wrapper library called `liburing` (created by Jens Axboe, original author of `io_uring`), which extracts most of the boilerplate away and provides convenient utilities for using `io_uring`.

We will now implement our program using `liburing`.

# Third Attempt: `liburing`

First of all we define a thin wrapper RAII class that creates the `io_uring` object and frees it upon destruction:

```cpp
class IOUring {
public:
  explicit IOUring(size_t queue_size) {
    if (auto s = io_uring_queue_init(NUM_FILES, &ring_, 0); s < 0) {
      throw std::runtime_error("error: " + std::to_string(s));
    }
  }

  IOUring(const IOUring &) = delete;
  IOUring &operator=(const IOUring &) = delete;
  IOUring(IOUring &&) = delete;
  IOUring &operator=(IOUring &&) = delete;

  ~IOUring() { io_uring_queue_exit(&ring_); }

  struct io_uring *get() {
    return &ring_;
  }

private:
  struct io_uring ring_;
};
```

The implementation consists of two parts: first we must push the file read requests to the submission queue, second we have to wait for them to arrive at the completion queue

```cpp
void pushEntriesToSubmissionQueue(const std::vector<ReadOnlyFile> &files,
                                  const std::vector<Buffer> &buffs,
                                  IOUring &uring) {
  for (size_t i = 0; i < files.size(); ++i) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(uring.get());
    io_uring_prep_read(sqe, files[i].fd(), buffs[i].get(), buffs[i].size(), 0);
    io_uring_sqe_set_data64(sqe, i);
  }
}
```
First of all we must ask the kernel to load the files. For this we create a `io_uring` instance, allocate the buffers, and write entries into the submission queue, one per file:

```cpp
std::vector<Result> iouring_only(const std::vector<ReadOnlyFile> &files) {
  auto buffs = initializeBuffers(files);

  IOUring uring{files.size()};
  for (size_t i = 0; i < files.size(); ++i) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(uring.get());
    io_uring_prep_read(sqe, files[i].fd(), buffs[i].get(), buffs[i].size(), 0);
    io_uring_sqe_set_data64(sqe, i);
  }
  // ...
}
```

`io_uring_get_sqe` creates an entry in the submission queue, `sqe`. We can now set the contents of the submission entry with `io_uring_prep_read`, with specifies that we want the kernel to read the contents of the file pointed to by `files[i].fd()` into the buffer `buffs[i]`.

One can also append user data to the submission entry with `io_uring_sqe_set_data`. This is data that is not used by the kernel but is just copied into the corresponding completion entry of the current request. This is important in order to differentiate which completion entry corresponds to which submission entry. In this case we just write the index of the file, which we can use to reference back when the read operation completes.

After we have written all the submission entries into the queue its time to submit them to the kernel and wait until they start appearing in the completion queue. Once completion entries appear, we can read the corresponding buffers into OBJ files:

```cpp
std::vector<Result> results;
while (results.size() < files.size()) {
  io_uring_submit_and_wait(uring.get(), 1);
  io_uring_cqe *cqe;
  unsigned head;
  int processed{0};
  io_uring_for_each_cqe(uring.get(), head, cqe) {
    auto id = io_uring_cqe_get_data64(cqe);
    results.push_back({.status_code = cqe->res, .file = files[id].path()});
    if (results.back().status_code) {
      readObjFromBuffer(buffs[id], results.back().result);
    }
    ++processed;
  }
  io_uring_cq_advance(uring.get(), processed);
}
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

# Todos

strace
github repo
reddit discussion

./run_all build_release
#########
Running normal threading
Processed 512 files.

real	0m0.383s
user	0m1.247s
sys	0m0.068s
#########
Running iouring
Processed 512 files.

real	0m0.352s
user	0m0.301s
sys	0m0.050s
#########
Running couroutines
Processed 512 files.

real	0m0.314s
user	0m0.293s
sys	0m0.020s
#########
Running coro pool
Processed 512 files.

real	0m0.218s
user	0m0.753s
sys	0m0.027s
#########
Running trivial
Processed 512 files.

real	0m0.324s
user	0m0.300s
sys	0m0.024s
