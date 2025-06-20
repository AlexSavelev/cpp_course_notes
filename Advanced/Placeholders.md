A placeholder type specifier designates a _placeholder type_ that will be replaced later, typically by deduction from an initializer.

# `auto`
- Since C++11
- Позволяет иногда не писать тип создаваемого объекта
- `for(auto it = map.begin(); it != map.end(); ++it) { ... }`
- `for(auto item : map) { ... }`

- Можно навешивать ссылки и константность
```cpp
const auto& y = x;
```

- Ссылки вообще всегда обрезаются
- `auto&&` работает как универсальная ссылка

### Rules
- Type is deduced using the rules for template argument deduction
	- `auto y = x;` is the same that `f(T value) <= f(x);`
- Отличие только одно: `auto` может выводить `std::initializer_list`
```cpp
template <typename T>
void foo(T arg) {}

int main() {
	foo({1, 2, 3});  // CE
	auto t = {1, 2, 3};  // std::inializer_list
}
```
- Also see DLI, CLI (C++17)

### `auto` в качестве возвращаемого значения
- Since C++14
- Очень полезно, когда тип большой или его нельзя выразить явно
	- Например, фабрика лямбда-функций
```cpp
auto f(int& x) {
	return x;
}

int main() {
	int v = 1;
	auto x = f(v);  // int
	auto& y = f(v);  // CE (lvalue link bind to rvalue int)
	auto&& z = f(v);  // int&&
}
```

```cpp
auto h(); // OK since C++14: h’s return type will be deduced when it is defined
```
#### Example
- Неоднозначность
	- Будет `CE`
	- Даже если можно неявно привести один тип к другому (никто не приоритизировал)
```cpp
auto f(int& x) {
	if (x > 0) {
		return x;
	}
	return 0u;
} // CE
```

### `auto` при `if constexpr`
- `if constexpr` разрешается до `auto`
```cpp
template <typename T>
auto f(int& x) {
  if constexpr (std::is_same_v<T, int>) {
    return x;
  } else {
    return 1.0;
  }
}
```

### trailing return type
- Синтаксис, позволяющий писать тип возвращаемого объекта по-другому
	- Полезно, когда этот тип зависит от аргументов
- Тут честно берется `int`
```cpp
auto f(int x) -> int {
	return x;
}
```

### `auto` в аргументах функций
- Since C++17
- Буквально сокращение шаблонов
	- То есть с `std::initializer_list` не прокатит
```cpp
#include <iostream>

void f(auto&& x) {  // сокращение шаблонов
  std::cout << __PRETTY_FUNCTION__ << '\n';
}

int main() {
  f(1);  // void f(auto:16&&) [with auto:16 = int]
  // f({1, 2, 3});  // CE
}
```

- Можно с вариадиками
```cpp
void f(auto... x) {

}
```

### `auto` как шаблонный аргумент
- Also see `auto` templates in Templates page

- Также можно создавать шаблонные переменные
- Since C++20
```cpp
#include <iostream>

template <auto X>
auto cringe = X;

int main() {
	std::swap(cringe<1>, cringe<2>);
}
```

### Conclusion example
```cpp
#include <iostream>
#include <string_view>

auto foo(int x) {}  // C++14

auto bar(auto y) {}  // C++17

template <auto X>
void baz() {}  // C++20

template <typename T>  // => <T, auto, auto>
void g(auto... x, T y) {
  std::cout << __PRETTY_FUNCTION__ << '\n';
}

template <auto X>
auto var = X;

void f(auto x) { std::cout << __PRETTY_FUNCTION__ << '\n'; }

template <typename T>
struct S {
  T x;
};

int main() {
  [](auto var) {};  // C++14

  auto x = 10;  // C++11
  bar<int>(10);

  g<int, double, double>(0, 0, 0);

  baz<2.2>();

  f(var<10>);
  f(var<2.2>);
  f(var<'a'>);

  std::cout << var<10> << '\n';
  var<10> = 100;
  std::cout << var<10> << '\n';
  std::swap(var<1>, var<2>);
  // var<S{10}>; CE ?! TODO
}
```


# `decltype`
- Since C++11
- Метафункция, которая в compile-time возвращает тип выражения
```cpp
int x;
decltype(x) u = x;
```
То есть `decltype` возвращает точный тип (с сохранением ссылок)

### 2 вида `decltype`
Есть 2 типа `decltype`:
- `decltype(statement)`
```cpp
int x;
decltype(x) y;  // statement - смотрим на тип объекта
// y = [int]
```
- `decltype(expression)`
```cpp
int x;
decltype((x)) y;  // expression
// y = [int&]
```
Вывод типов в `decltype(expr)` работает по следующим правилам:
1. if the value category of expression is xvalue, then `decltype` yields `T&&`;
	- `decltype(std::move(x))`
2. if the value category of expression is lvalue, then `decltype` yields `T&`;
	- `decltype(++x)`
3. if the value category of expression is prvalue, then `decltype` yields `T`.
	- `decltype(x++)`

### Навешивание на `decltype`
- Можно также всякое навешивать
```cpp
int x;
decltype(x)& y = x;
decltype(x)* yy = &x;
decltype(x)&& = std::move(x);  // не универсальная ссылка
```

### Example 1
```cpp
template <typename T>
void f() = delete;

int main() {
	int x;
	int& y = x;
	const decltype(y) z = x;
	f<decltype(z)>(); // T = T&
}
```
- Это происходит потому что `const` навешивается не на то, что лежит под ссылкой, а на саму ссылку. А ссылки и так нельзя перевешивать (т.е. они и так константные)

### Example 2
- Код внутри `decltype` не выполняется - это compile-time конструкция
	- Он просто возьмет из сигнатуры выражения возвращаемый тип
```cpp
int x = 0;
decltype(++x) y = x;  // y = [int&]
++y;

std::cout << x << y;  // 11
```

### Example 3
```cpp
decltype(throw 1)* p = nullptr;  // void* p = nullptr;
```

### `decltype` в качестве возвращаемого значения
- Хотим всегда возвращать тип такой, какой вернет `some_func`, но допустим что `some_func` умеет возвращать ссылку или не ссылку для некоторых типов
- Поэтому `auto` не поможет (срезает ссылки)
```cpp
template <typename T>
... g(const T& x) {
	return some_func(x);
}
```

- `decltype` тоже не поможет, тк `x` еще не объявлен
```cpp
template <typename T>
decltype(some_func(x)) g(const T& x /* point of decl. here */) {
  return some_func(x)
}
```

- Solution - trailing return type
```cpp
template <typename T>
auto g(const T& x) -> decltype(some_func(x)) {
	return some_func(x);
}
```

### Example 4
- Не всегда обращение к контейнеру по индексу возвращает ссылку (`std::vector<bool>`), поэтому приходится вот так извращаться с возвращаемым типом
```cpp
template <typename Container>
auto get(const Container& c, size_t i) -> decltype(c[i]) {
  return c[i];
}
```

### `decltype(auto)`
- С C++14
- Механизм выводить тип не по правилам `auto`, а по правилам `decltype`
	- `statement`/`expression` также определяется

```cpp
template <typename Container>
decltype(auto) get(const Container& c, size_t i) {
  return c[i];
}
```

```cpp
int x = 0;
int& y = x;
decltype(auto) z = y;  // int&
```

# Structured bindings
- С C++17
```cpp
int main() {
	std::pair<int, std::string> p(5, "abc");
	auto [a, b] = p;
	std::cout << a << ' ' << b << '\n';
}
```
- Работает для tuple-like, C'шных массивов, data members

```cpp
struct Test { ... };

int main() {
	Test t;
	auto& [s1, s2, a, v1] = t;
}
```

### Example
```cpp
struct Test {
	std::vector<int> v1;
	std::string s1;
	std::deque<int> d1;
	std::map<int, int> m1;

	bool IsEmpty() const {
		// return v1.empty() && s1.empty() && d1.empty() && m1.empty(); // best solution
		auto& [a1, a2, a3, a4] = *this;
		return [](auto&&... args) -> bool { return (args.empty() && ...); }(a1, a2, a3, a4);
	}
};
```

# Binding process
- [Source](https://en.cppreference.com/w/cpp/language/structured_binding)

- For initializer syntax `... = expression`, the elements are copy-initialized
- For initializer syntaxes `...()` or `...{}`, the elements are direct-initialized

We use `E` to denote the type of the identifier expression `e` (i.e., `E` is the equivalent of `std::remove_reference_t<decltype((e))>`).

A _structured binding size_ of `E` is the number of structured bindings that need to be introduced by the structured binding declaration.

A structured binding declaration performs the binding in one of three possible ways, depending on `E`:
- Case 1: If `E` is an array type, then the names are bound to the array elements.
- Case 2: If `E` is a non-union class type and `std::tuple_size<E>` is a complete type with a member named `value` (regardless of the type or accessibility of such member), then the "tuple-like" binding protocol is used.
- Case 3: If `E` is a non-union class type but `std::tuple_size<E>` is not a complete type, then the names are bound to the accessible data members of `E`.

### Case 1: binding an array
Each structured binding in the sb-identifier-list becomes the name of an lvalue that refers to the corresponding element of the array. The structured binding size of `E` is equal to the number of array elements.

The _referenced type_ for each structured binding is the array element type. Note that if the array type `E` is cv-qualified, so is its element type.

```cpp
int a[2] = {1, 2};
 
auto [x, y] = a;    // creates e[2], copies a into e,
                    // then x refers to e[0], y refers to e[1]
auto& [xr, yr] = a; // xr refers to a[0], yr refers to a[1]
```

### Case 2: binding a type implementing the tuple operations
The expression `std::tuple_size<E>::value` must be a well-formed integral constant expression, and the structured binding size of `E` is equal to `std::tuple_size<E>::value`.

For each structured binding, a variable whose type is "reference to `std::tuple_element<I, E>::type`" is introduced: lvalue reference if its corresponding initializer is an lvalue, rvalue reference otherwise. The initializer for the Ith variable is:
- `e.get<I>()`, if lookup for the identifier `get` in the scope of `E` by class member access lookup finds at least one declaration that is a function template whose first template parameter is a non-type parameter
- Otherwise, `get<I>(e)`, where get is looked up by argument-dependent lookup only, ignoring non-ADL lookup.

In these initializer expressions, e is an lvalue if the type of the entity e is an lvalue reference (this only happens if the ref-qualifier is `&` or if it is `&&` and the initializer expression is an lvalue) and an xvalue otherwise (this effectively performs a kind of perfect forwarding), I is a `std::size_t` prvalue, and `<I>` is always interpreted as a template parameter list.

The structured binding then becomes the name of an lvalue that refers to the object bound to said variable.

The _referenced type_ for the Ith structured binding is `std::tuple_element<I, E>::type`.

### Case 3: binding to data members
Every non-static data member of `E` must be a direct member of `E` or the same base class of `E`, and must be well-formed in the context of the structured binding when named as `e.name`. `E` may not have an anonymous union member. The structured binding size of `E` is equal to the number of non-static data members.

Each structured binding in sb-identifier-list becomes the name of an lvalue that refers to the next member of `e` in declaration order (bit-fields are supported); the type of the lvalue is that of `e.mI`, where `mI` refers to the `I`th member.

The _referenced type_ of the `I`th structured binding is the type of `e.mI` if it is not a reference type, or the declared type of `mI` otherwise.

```cpp
#include <iostream>
 
struct S {
    mutable int x1 : 2;
    volatile double y1;
};
 
S f() { return S{1, 2.3}; }

int main() {
    const auto [x, y] = f(); // x is an int lvalue identifying the 2-bit bit-field
                             // y is a const volatile double lvalue
    std::cout << x << ' ' << y << '\n';  // 1 2.3
    x = -2;   // OK
//  y = -2.;  // Error: y is const-qualified
    std::cout << x << ' ' << y << '\n';  // -2 2.3
}
```
#### Initialization order
Let `valI` be the object or reference named by the `I`th structured binding in sb-identifier-list ﻿:
- The initialization of `e` is sequenced before the initialization of any `valI`.
- The initialization of each `valI` is sequenced before the initialization of any `valJ` where `I` is less than `J`.

# Adding structured binding support to custom types
- [Source](https://devblogs.microsoft.com/oldnewthing/20201015-00/?p=104369)
- Example for:
```cpp
class Person {
 public:
  std::string name;
  int age;
};
```

1) Include `<utility>`.
2) Specialize the `std::tuple_size` so that its `value` is a `std::size_t` integral constant that says how many pieces there are.
```cpp
namespace std {
  template<>
  struct tuple_size<::Person> {
    static constexpr size_t value = 2;
  };
}
```

```cpp
namespace std {
  template<>
  struct tuple_size<::Person>
      : integral_constant<size_t, 2> {};
}
```

3) Specialize the `std::tuple_element` so that it identifies the type of each piece. You need as many specializations as you have pieces you declared in Step 2. The indices start at zero.
```cpp
namespace std {
  template<>
  struct tuple_element<0, ::Person> {
    using type = std::string;
  };

  template<>
  struct tuple_element<1, ::Person> {
    using type = int;
  };
}
```

```cpp
namespace std {
  template<size_t Index>
  struct tuple_element<Index, ::Person>
    : conditional<Index == 0, std::string, int> {
    static_assert(Index < 2,
      "Index out of bounds for Person");
  };
}
```

```cpp
namespace std {
  template<size_t Index>
  struct tuple_element<Index, ::Whatever>
    : tuple_element<Index, tuple<std::string, int, whatever>> {};
}
```

4) Provide all of the `get` functions.
```cpp
class Person {
 public:
  std::string name;
  int age;

  template<std::size_t Index>
  auto&& get()       &  { return get_helper<Index>(*this); }

  template<std::size_t Index>
  auto&& get()       && { return get_helper<Index>(*this); }

  template<std::size_t Index>
  auto&& get() const &  { return get_helper<Index>(*this); }

  template<std::size_t Index>
  auto&& get() const && { return get_helper<Index>(*this); }

 private:
  template<std::size_t Index, typename T>
  auto&& get_helper(T&& t) {
    static_assert(Index < 2, "Index out of bounds for Custom::Person");
    if constexpr (Index == 0) return std::forward<T>(t).name;
    if constexpr (Index == 1) return std::forward<T>(t).age;
  }
};
```

```cpp
template<std::size_t Index, typename T>
auto&& Person_get_helper(T&& p) {
  static_assert(Index < 2,
    "Index out of bounds for Custom::Person");
  if constexpr (Index == 0) return std::forward<T>(t).name;
  if constexpr (Index == 1) return std::forward<T>(t).age;
}

template<std::size_t Index>
auto&& get(Person& p) {
  return Person_get_helper<Index>(p);
}

template<std::size_t Index>
auto&& get(Person const& p) {
  return Person_get_helper<Index>(p);
}

template<std::size_t Index>
auto&& get(Person&& p) {
  return Person_get_helper<Index>(std::move(p));
}

template<std::size_t Index>
auto&& get(Person const&& p) {
  return Person_get_helper<Index>(move(p));
}
```

### Result
```cpp
Person p;

auto&& [name, age] = p;
name = "Fred";
age = 42;
```

# Examples
### Disable compiler caching
```cpp
template <auto t = [](){}>
constexpr int f() { return 1 + 1; }

int main() {
	f();
	f();
	f();  // Disable caching

	return 0;
}
```

```cpp
template <auto = []() {}>
struct S {
  S() = default;
  ~S() = default;
};

int main() {
  S s1, s2;
  S s3;
}

```
- See also about [loopholes](https://stackoverflow.com/questions/65190015/c-type-loophole-explanation) (please don't do this)
