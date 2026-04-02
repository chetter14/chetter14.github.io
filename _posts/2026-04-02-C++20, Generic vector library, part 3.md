---
layout: post
title: C++20. Generic vector library. Part 3.
---

I decided to update the project slightly by improving its *code quality* and adding *CMake* and *GTest*. Once those updates are complete, I’ll consider this learning project finished.

I won’t dive into every change I made to the code. Instead, I’ll just highlight a few updates that I think *are worth mentioning*

1) `noexcept` **specifiers**. Several methods in the *Vector* class are marked with a conditional `noexcept` specifier. This specifier evaluates a `noexcept` expression to determine whether the function should be treated as `noexcept`. Example:
```
constexpr Vector& operator+=(const Vector& v) noexcept(
    noexcept(std::declval<T&>() += std::declval<T&>())) {
  ...
}
```

This ensures the `noexcept` status dynamically evaluates to `true` or `false` depending on whether the underlying operations on the stored type (`T`) are themselves `noexcept`.

2) **STL algorithms**. I also replaced all manual `for` loops with standard library algorithms. This aligns with the "no raw loops" principle popularized by *Sean Parent*. I didn’t adopt it just to follow a rule - I genuinely believe this approach leads to cleaner code, making it more *readable and modular*.

```
constexpr Vector& operator+=(const Vector& v) noexcept(
    noexcept(std::declval<T&>() += std::declval<T&>())) {
  std::ranges::transform(m_arr, v, m_arr.begin(),
                          [](const auto& a, const auto& b) { return a + b; });
  return *this;
}
...
constexpr Vector operator-() const {
  Vector res = *this;
  std::for_each(res.begin(), res.end(), [](auto& val) { val = -val; });
  return res;
}
```

3) **Printing the `Vector` object**. I removed the `friend`-qualified `operator<<` from the `Vector` class. Instead, I added a `dump` method and defined a global `operator<<` for `Vector` that simply calls it:
```
class Vector {
  ...
  void dump(std::ostream& os) const {
   std::for_each(m_arr.begin(), m_arr.end(),
                 [&](auto val) { os << val << " "; });
 }
 ...
};

export template <std::size_t N, VectorElementType T>
std::ostream& operator<<(std::ostream& os, const Vector<N, T>& v) {
  v.dump(os);
  return os;
}
```
This approach ensures that arbitrary `friend` functions cannot directly access or manipulate the class's internal state. Instead, output functionality is strictly confined to a dedicated, controlled method.

**CMake**. I created a `CMakePresets.json` file with a configuration tailored for Visual Studio 2022. This was primarily for my own workflow, as I developed the project using that IDE.

**GTest**. I implemented a basic test suite (`static_assert` checks and a few initialization cases) to verify that GTest integrates properly and that tests run smoothly via *CTest*. In a real-world scenario, I would write comprehensive tests covering all `Vector` logic.

With that, I consider the project complete. I achieved my learning goals and don’t see a need to extend it further. While there’s certainly room for improvement and new features, this was always intended as a sandbox rather than a production-ready library. I already have ideas for more complex and interesting projects, and I’ll be sure to share updates once I start building them.