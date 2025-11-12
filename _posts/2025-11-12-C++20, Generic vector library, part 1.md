---
layout: post
title: C++20. Generic vector library. Part 1.
---

Recently, I finally decided to get my hands on *C++20*. I began reading *A Tour of C++* (3rd edition) by Bjarne Stroustrup. After reading more than half of it, I can say that it’s pretty good — you can definitely find some cool new things for yourself.

But reading alone isn’t enough, so I decided to complete a project to apply the knowledge and skills I’ve learned in real-world, concrete cases.

The first such project is a **generic vector library**. It’s a header-only library that defines **operations on mathematical vectors such as** addition, subtraction, multiplication, calculating magnitude, accessing elements, and more. This list isn’t final and may be extended later (I might want to add more functionality).

The project is available [here](https://github.com/chetter14/generic-vector-library).

The main idea of this project is to *deal with concepts, type functions, and metaprogramming* in general. So the implementation of the library is not the core, but the interface is.

Before diving into the library itself, I should say that I use modules in this project as they were added in C++20. And at the moment library is *placed in .cppm file*. Later I'll add a version of it in a header.

The main idea of this project is to **experiment with concepts, type functions, and metaprogramming** in general. So, the core of the project isn’t the implementation itself, but rather the interface design.

Before diving into the library itself, I should mention that I’m using **modules** in this project, as they were introduced in C++20. At the moment, the library is **contained in a** `.cppm` **file**. Later, I’ll also provide a header-based version.

At first, I wrote a *Makefile* to build both the library and a sample program that uses it.
Currently, the C++ compiler version and compilation flags are as follows:
```
CXX = /opt/homebrew/opt/llvm/bin/clang++
CXXFLAGS = -std=c++20 -stdlib=libc++
```
The C++ compiler path is hardcoded for my local development setup for now — I’ll fix this later.

One more interesting part of the *Makefile* is the additional compilation options:

1) `-fmodule-output=Vector.pcm` - used when building the `.cppm` file (similar to a traditional header) to specify the name of the **Compiled Module Interface (CMI)**, which will then be used when building `.cpp` files such as the sample.
2) `-fprebuilt-module-path=.` - specifies the location of the CMI, allowing the sample (or any other file) to *import* the module and use its functionality.

In short, it means:
> “You have a `Vector.cppm` file — compile it to produce a `Vector.pcm` file. Then, use `Vector.pcm` when compiling `main.cpp`.”

An important note: *currently*, there are no concepts, template constraints, or similar features yet. I first wanted to build a basic `Vector` class with simple logic, and only after that add template parameters and apply metaprogramming principles.

Now, onto the `Vector` **class module**.
At the top, I declare a *global module fragment* to include the standard C++ headers used inside my exported module. After that, the `Vector` class itself is defined:

```
module;
#include <array>
#include <algorithm>
#include <iostream>
#include <cmath>
#include <ranges>
#include <type_traits>

export module Vector;

export template<int N, typename T>
class Vector
{
    ...
};
```

I’ve defined an **initializer-list constructor** for `Vector` objects.
The copy/move constructors, destructor, and assignment operators are all defaulted since the `std::array` member doesn’t require any special handling.
```
Vector(std::initializer_list<T> lst) 
{
    std::ranges::copy(lst, m_arr.begin());
}
// ... default constructors, destructor, and assignments
```

I’ve also provided `begin()` and `end()` functions so that range-based for loops can be used with the `Vector` object:
```
// for modification:
auto begin() noexcept { return m_arr.begin(); }
auto end() noexcept { return m_arr.end(); }

// for read-only access:
auto begin() const noexcept { return m_arr.begin(); }
auto end() const noexcept { return m_arr.end(); }
```

Next, I declared **operator overloading** functions and implemented the **magnitude calculation**. Their definitions are straightforward and not particularly interesting to dive into here.

I also defined other operations for the `Vector` class — such as **addition**, **subtraction**, **inversion**, and **printing** the vector contents.

The `main.cpp` file contains simple test logic to verify that various operations on `Vector` objects work correctly and that no errors occur during **compilation**, **linking**, or **runtime**.

That’s it for the initial implementation of the `Vector` class.

The next step is to integrate **concepts**, **requires-clauses**, and other **metaprogramming techniques** into the class to make it more generic and expressive.