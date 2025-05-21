
# 0. Переопределение ключевых слов
- Управляем доступом к библиотеке без её изменения
```cpp
#define public private
#include "mylibrary.h"
#undef private
```

- Или просто hot-fix 01/04/2025
```cpp
#define true false
#define else
#define int float
#define float char
```


# 1. When is this function safe or unsafe to use?
```cpp
template<auto V>
const auto& foo() { return V; }
```
### Answer
- It's safe for class types and unsafe for, say, `int`'s. For some reason the standard threats them differently, so:
```cpp
const auto& v1 = foo<42>();       // bad! dangling reference

struct S { int val; };
const auto& v2 = foo<S { 42 }>(); // fine!
```

# 2. How are these two functions different?
```cpp
template<typename T>
T mkT1() { return {}; }
   
template<typename T>
T mkT2() { return T {}; }
```
### Answer
Besides the obvious difference in handling of `explicit` vs `nonexplicit` default constructors, consider `std::mutex` and C++14 vs C++17.

```cpp
template <typename T>
T mkT1() {
  return {};
}

template <typename T>
T mkT2() {
  return T{};
}

struct Explicit { explicit Explicit() {} };
struct Implicit { Implicit() {} };

int main() {
  mkT1<Explicit>();  // CE: converting to ‘Explicit’ from initializer list would use explicit constructor ‘Explicit::Explicit()’
  mkT2<Explicit>();  // OK

  mkT1<Implicit>();  // OK
  mkT2<Implicit>();  // OK
}
```

These have _almost_ exactly the same effect:
- `T x = { 1, 2, 3 };`
- `T x { 1, 2, 3 };`
Technically the version with `=` is called _copy-list-initialization_ and the other version is _direct-list-initialization_ but the behaviour of both of those forms is specified by the _list-initialization_ behaviour.

The differences are:
- If _copy-list-initialization_ selects an `explicit` constructor then the code is ill-formed (as stated in 13.3.1.7 `[over.match.list]`)
- If `T` is `auto`, then:
    - _copy-list-initialization_ deduces `std::initializer_list<Type_of_element>`
    - _direct-list-initialization_ only allows a single element in the list, and deduces `Type_of_element`.

`T obj = T{...}` (excluding `auto`) is exactly the same as `T obj{...}`, since C++17, i.e. _direct-list-initialization_ of `obj`. Prior to C++17 there was _direct-list-initialization_ of a temporary, and then copy-initialization of `obj` from the temporary.

This is a true elision, whereby just writing `T{}` doesn't really create a `T`, but instead says "I want a `T`", and an actual temporary is "materialised" only if/when it needs to be.

Redundant utterances of this fact are effectively collapsed into one, so despite screaming "I want a `T`! I want a `T`! I want a `T`! I want a `T`!" the child still only gets one `T` in the end.

So, in C++17, `T obj = T{...}` is literally equivalent to `T obj{...}`.

- [See more about copy elision](https://en.cppreference.com/w/cpp/language/copy_elision)
> In the initialization of an object, when the initializer expression is a prvalue of the same class type (ignoring cv-qualification) as the variable type _[..]_

- [Source](https://stackoverflow.com/questions/60805366/object-initialization-syntax-in-c-t-obj-vs-t-obj)

==TODO== Add [About {}()](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#es23-prefer-the--initializer-syntax)
- About auto CLI  ==TODO== CE about
```cpp
#include <iostream>

template <typename T>
void foo(T args) {
  // void foo(T) [with T = std::initializer_list<int>]
  std::cout << __PRETTY_FUNCTION__ << '\n';
}

int main() {
  // foo({1, 2, 3});  // CE

  auto a = {1, 2, 3};
  foo(a);
}
```

```cpp
#include <iostream>
#include <vector>

struct S {
  S(int x, int y) { std::cout << "S(x, y)" << '\n'; }

  S(std::initializer_list<int>) { std::cout << "S(il)" << '\n'; }

  S(double x, double y) {}
};

template <typename T, typename... Args>
void ConstructAt(T* ptr, Args&&... args) {
  new (ptr) T(std::forward<Args>(args)...);
}

int main() {
  std::vector v1(10, 20);  // {20 20 20 20 20 20 20 20 20 20}
  std::vector v2{10, 20};  // {10 20}
                           //
  S s1(10, 20);
  S s2{10, 20};

  std::vector<int> v3{v1.begin(), v1.end()};
  char buffer[24];
  std::construct_at(reinterpret_cast<std::vector<int>*>(buffer), 10, 20);
  std::vector v4{v3};

  // S s3{2.2, 3.3}; CE
}
```

# 3. Is this code valid?
```cpp
struct Foo {
  int a;

  Foo() = delete;
  Foo(int) = delete;
};
   
int main() {
  Foo foo1 {};
  Foo foo2 { 10 };
}
```

### Answer
Depends on the C++ version.
Up until C++17, both variables are initialized with aggregate initialization. `Foo foo` and `Foo foo(10)` wouldn't be valid, though.
Starting with C++20, this somewhat counter-intuitive behaviour is fixed, and this code no longer compiles.

# 4. (^_=...=_^)
1. What does this code do, and on what features of C++17 does it rely?
2. What problems does this code have, and what should be done to fix them?
```cpp
template<typename F, typename... Ts>
void foo(F f, Ts... ts) {
  int _;
  (_ = ... = (f(ts), 0));
}
```

### Answer
1. It calls the function on the elements of the variadic pack in reverse order.
2. `f` might return something with an overloaded `operator,`
	- Expression is not assignable (`0 = 0`)

# 5. I C `memset`
Assume an instance of a `struct` is `memset`ed to zeroes. What would be the value of the padding?
Further assume a field of that structure is updated. What would be the value of the padding after that field? After other fields?

### Answer
Unspecified, unspecified

# 6. Is this code valid?
```cpp
char arr[5] = { 0 };
auto pastEnd = arr + 10;
```

```cpp
char arr[5] = { 0 };
auto pastEnd = arr + 5;
```
### Answer
No, no - segmentation fault

# 7. Which lines are UB, if any?
```cpp
#include <iostream>

struct Foo1 {
    int a;
};

struct Foo2 {
    int a;
    Foo2() = default;
};

struct Foo3 {
    int a;
    Foo3();
};

Foo3::Foo3() = default;

int main() {
    Foo1 foo11, foo12 {};
    Foo2 foo21, foo22 {};
    Foo3 foo31, foo32 {};

    std::cout << foo11.a << std::endl;
    std::cout << foo12.a << std::endl;
    std::cout << foo21.a << std::endl;
    std::cout << foo22.a << std::endl;
    std::cout << foo31.a << std::endl;
    std::cout << foo32.a << std::endl;
}
```
==TODO== ans

# 8. Is this code valid?
```cpp
struct X { int a, b; };

X *make_x() {
    X *p = (X*)malloc(sizeof(struct X));
    p->a = 1;
    p->b = 2;
    return p;
}
```

### Answer
Depends on the C++ version, and whether it is C++ to begin with.
Up until C++17, neither an `x` object nor an `int` subobjects are created, and this code is UB.
Starting with C++20, an `x` object and its `int` subobjects are implicitly created, and this code is valid.
It always has been valid C code, though.

- Функция `malloc` возвращает нулевой указатель, если невозможно выделить буфер памяти указанного размера. Поэтому прежде, чем разыменовать указатель, его нужно проверить на равенство `NULL`
- А так это минимальное, о чем стоит беспокоиться при использовании `malloc`

# 9. Is using this function dangerous?
```cpp
auto foo1() {
    return "Gotta love C++";
}
```

What about this one?
```cpp
auto foo2() {
    const char *str = "Gotta love C++";
    return str;
}
```

This one?
```cpp
auto foo3() {
    const char str[] = "Gotta love C++";
    return str;
}
```

### Answer
1. NO
2. NO
3. YES:
```OUT
warning: address of local variable ‘str’ returned
```

### Question 2
Why? What's the crucial difference between these functions? Is there any difference in their types?
### Answer
Accessing any element of the "array" returned by `foo1` and `foo2` is fine. Try doing that to `foo3` and you'll get an UB, since you'll be using an object whose lifetime has ended!
`foo1` and `foo2` return a pointer to a string that is, roughly speaking, allocated and stored somewhere in the executable at compile time.
The pointer returned by `foo3` references the _local_ array `str` which is initialized by _copying_ that same string. This array is local to `foo3` and its lifetime ends once the function has returned, hence the UB.

### Question 3
While modern compilers output a warning, what's a reliable and somewhat general way to check functions like this?

### Answer
Mark all these functions `constexpr` and try using them in a constant evaluated context, like `static_assert`:

```cpp
static_assert(foo1()[0] == 'G');
static_assert(foo2()[0] == 'G');
static_assert(foo3()[0] == 'G');
```

Clang outputs:
```OUT
error: non-constant condition for static assertion
   22 | static_assert(foo3()[0] == 'G');
      |               ~~~~~~~~~~^~~~~~
error: accessing 'str' outside its lifetime
   22 | static_assert(foo3()[0] == 'G');
      |               ~~~~~~~~^
note: declared here
   16 |     const char str[] = "Gotta love C++";
      |                ^~~
```

# 10. Fun with fun templates
What does it print?
```cpp
#include <iostream>

template<typename T>
int foo(T) { return 1; }

template<>
int foo(int*) { return 2; }

template<typename T>
int foo(T*) { return 3; }

int main() {
    int test;
    std::cout << foo(&test) << foo<int>(&test) << foo<int*>(&test) << '\n';
}
```

What if we reorder the definitions?
```cpp
#include <iostream>

template<typename T>
int foo(T) { return 1; }

template<typename T>
int foo(T*) { return 3; }

template<>
int foo(int*) { return 2; }

int main() {
    int test;
    std::cout << foo(&test) << foo<int>(&test) << foo<int*>(&test) << '\n';
}
```

Can we still specialize the first template after we've introduced the second one?
### Answer
1. `332`
2. `221`
3. Yep:
```cpp
template<>
int foo<int*>(int*) { return 2; }
```

# 11. Our favorite ranges
```cpp
struct Evil {
  auto begin() { return std::counted_iterator("Library", 7); }
  friend auto begin(Evil&) { return std::counted_iterator("Core", 4); }
  friend auto end(Evil&) { return std::default_sentinel; }
};

Evil rg;
for (char c : rg) { putchar(c); }
std::ranges::for_each(rg, [](char c) { putchar(c); });
```
==TODO==
