---
layout: post
title: C++20. Generic vector library. Part 2.
---

Now we come to an interesting topic: **concepts**. I'm going to delve into all the concepts I have defined and describe what each one does and why it is needed.

First, our vector can only be of size 1, 2, or 3—no less than 1, and no more than 3. This is enforced by the following concept:

```
export template <std::size_t N>
concept CorrectVectorSize = (N >= 1 && N <= 3);
```

> **Explanation**: The `export` keyword before `template` means that the concept is exported and can be used in other modules that import this one.

Second, vector values are inverted in the `invert()` function, so we must ensure that the value type supports this operation:
```
export template <typename T>
concept Invertable = requires(T x) {
  -x;
};
```

I have overloaded the operators `+=`, `+`, `-=`, and `-`. Since they operate on vector values, I defined a concept for them:
```
export template <typename T, typename U>
concept Arithmetic = requires(T x, U y) {
  x + y;
  x - y;
  x += y;
  x -= y;
};
```

To print out vector values:
```
export template <typename T>
concept Streamable = requires(T x, std::ostream& os) {
  os << x;
};
```

And also to compare vector values with each other and to move them:
```
export template <typename T>
concept Comparable = requires(T x, T y) {
  x == y;
  x != y;
  x > y;
  x >= y;
  x < y;
  x <= y;
};

export template <typename T>
concept Movable = requires(T x, T y) {
  x = std::move(y);
  T{std::move(x)};
};
```

Ultimately, the vector's element type must satisfy all of the specifications above. This is combined into a single concept:
```
export template <typename T>
concept VectorElementType =
    std::regular<T> && Invertable<T> && Arithmetic<T, T> && Streamable<T> &&
    Comparable<T> && Movable<T>;
```

> **Explanation**: `std::regular` is a type trait that checks if a type has a default constructor, copy constructor, and copy assignment operator.

Now, in the `Vector` class definition, we use these concepts as constraints:
```
export template <std::size_t N, VectorElementType T>
requires CorrectVectorSize<N> class Vector {
  // ...
};
```

I decided to make the scaling operation in my `Vector` class only possible with integer or floating-point numbers. This is enforced by a simple concept:
```
export template <typename T>
concept RealType = std::integral<T> || std::floating_point<T>;
```

Later, I defined a more specific concept, `ScalableWith`, and used it in conjunction with `RealType` to constrain `operator*=`:
```
export template <typename T, typename U>
concept ScalableWith = requires(T elem, U scalar) {
  elem *= scalar;
};

...

class Vector { 
  ... 
  constexpr Vector& operator*=(RealType auto scalar) noexcept requires
      ScalableWith<T, decltype(scalar)> {
    for (auto& val : m_arr) {
      val *= scalar;
    }
    return *this;
  }
  ... 
};
```

For the operators `+`, `-`, `+=`, and `-=`, I used the `Arithmetic` concept to ensure that the values of both `Vector`s can be added or subtracted. I also implemented these operators efficiently by defining `operator+` in terms of `operator+=`:
```
  template <typename U>
  constexpr Vector& operator+=(const Vector<N, U>& v) noexcept requires
      Arithmetic<T, U> {
    for (int i = 0; i < N; ++i) {
      m_arr[i] += v[i];
    }
    return *this;
  }

  template <typename U>
  friend constexpr Vector operator+(const Vector& lhs,
                                    const Vector<N, U>& rhs) noexcept requires
      Arithmetic<T, U> {
    Vector res = lhs;
    res += rhs;
    return res;
  }
  // The same pattern is used for operator- and operator-=
```

In the `main.cpp` file, I added a series of static assertions to verify that the concepts work as intended:
```
static_assert(RealType<float>);
static_assert(RealType<int>);
static_assert(RealType<double>);
static_assert(!RealType<std::string>);
static_assert(!RealType<std::tuple<double, int>>);
// Other static_assert's with concepts
```

Additionally, in `main.cpp`, I wrote several logical operations on `Vector` objects to demonstrate their functionality and usage.

I believe I have successfully implemented everything I set out to do, focusing on **concepts and template constraints**. I also successfully integrated **C++20 modules**, which is a great achievement.

There is certainly room for improvement and further modifications - such as adding a `Matrix` class composed of `Vector` objects or extending the set of operations on `Vector`. However, for now, I consider this project complete.

