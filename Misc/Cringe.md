- [Source](https://github.com/0xd34df00d/you-dont-know-cpp) ==TODO== r on release

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
==TODO== ans

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
Up until C++17, both variables are initialized with aggregate initialization. `Foo foo` and `Foo foo(10)` wouldn't be valid, though. ==TODO== Why?
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
==TODO== operator, ??

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
==TODO== WHY?

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

# 11. What does this print?
- [Source](https://quuxplusone.github.io/blog/2024/12/09/foreach-versus-for/)
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
==TODO== check
