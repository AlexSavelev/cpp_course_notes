# Types

## Common operators

| Assignment                                                                                                                         | Inc/Dec                          | Arithmetic                                                                                                                            | Logical                                                                            | Comparison                                                                      | Member access                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `a = b`<br>`a += b`<br>`a -= b`<br>`a *= b`<br>`a /= b`<br>`a %= b`<br>`a &= b`<br>`a \|= b`<br>`a ^= b`<br>`a <<= b`<br>`a >>= b` | `++a`<br>`--a`<br>`a++`<br>`a--` | `+a`<br>`-a`<br>`a + b`<br>`a - b`<br>`a * b`<br>`a / b`<br>`a % b`<br>`~a`<br>`a & b`<br>`a \| b`<br>`a ^ b`<br>`a << b`<br>`a >> b` | `!a`<br>`a && b`<br>`a \|\| b`<br>(не перегружай - потеряешь ленивость вычислений) | `a == b`<br>`a != b`<br>`a < b`<br>`a > b`<br>`a <= b`<br>`a >= b`<br>`a <=> b` | `a[...]`<br>`*a`<br>`&a`<br>`a->b`<br>`a.b`<br>`a->*b`<br>`a.*b` |
- Overloading unary minus and plus is not very common and probably best avoided. If needed, they should probably be overloaded as member functions.
- If you provide `+`, also provide `+=`, if you provide `-`, do not omit `-=`, etc.

- Other:

| Function call | Comma  | Conditional |
| ------------- | ------ | ----------- |
| `a(...)`      | `a, b` | `a ? b : c` |

## Special operators
- `static_cast` converts one type to another related type  
- `dynamic_cast` converts within inheritance hierarchies  
- `const_cast` adds or removes cv-qualifiers
- `reinterpret_cast` converts type to unrelated type
- `C-style cast` converts one type to another by a mix of static_cast, const_cast, and reinterpret_cast
- `new` creates objects with dynamic storage duration
- `delete` destructs objects previously created by the new expression and releases obtained memory area
- `sizeof` queries the size of a type
- `sizeof...` queries the size of a pack (since C++11)
- `typeid` queries the type information of a type
- `noexcept` checks if an expression can throw an exception (since C++11)
- `alignof` queries alignment requirements of a type (since C++11)

- [Source 1](https://stackoverflow.com/questions/4421706/what-are-the-basic-rules-and-idioms-for-operator-overloading#4421719)
- [Source 2](https://en.cppreference.com/w/cpp/language/operators)

`std::cout << x = 5; // 5`
- С C++17 правая часть operator= вычисляется первой (`++x = x++`)

## **List of operators that cannot be overloaded**

| No  | Operator           | Name                            |
| --- | ------------------ | ------------------------------- |
| 1   | `::`               | Scope Resolution Operator       |
| 2   | `?:`               | Ternary or Conditional Operator |
| 3   | `.`                | Member Access or Dot operator   |
| 4   | `.*`               | Pointer-to-member Operator      |
| 5   | `sizeof`           | Object size Operator            |
| 6   | `typeid`           | Object type Operator            |
| 7   | `static_cast`      | casting operator                |
| 8   | `const_cast`       | casting operator                |
| 9   | `reinterpret_cast` | casting operator                |
| 10  | `dynamic_cast`     | casting operator                |

# Overloading methods

| Expr      | As member function     | As non-member function |
| --------- | ---------------------- | ---------------------- |
| `@a`      | `(a).operator@ ( )`    | `operator@ (a)`        |
| `a @ b`   | `(a).operator@ (b)`    | `operator@ (a, b)`     |
| `a = b`   | `(a).operator= (b)`    | cannot be non-member   |
| `a(b...)` | `(a).operator()(b...)` | cannot be non-member   |
| `a[b...]` | `(a).operator[](b...)` | cannot be non-member   |
| `a->`     | `(a).operator->( )`    | cannot be non-member   |
| `a@`      | `(a).operator@ (0)`    | `operator@ (a, 0)`     |
- In this table, `@` is a placeholder representing all matching operators: all prefix operators in `@a`, all postfix operators other than `->` in `a@`, all infix operators other than `=` in `a@b`.

# Operator precedence
- [Article](https://en.cppreference.com/w/cpp/language/operator_precedence)

# Spaceship operator
```cpp
#include <compare>
#include <ios>
#include <iostream>

struct Int {
  int x;
  int y;

  std::strong_ordering operator<=>(const Int& other) const = default;
  // bool operator==(const Int& other) const = default;
  friend bool operator==(const Int&, const Int&) = default;
};

// bool operator==(const Int&, const Int&) {}

int main() {
  Int x{1, 1},
      y{2, 2},
      z{1, 2};
  std::cout << std::boolalpha;
  std::cout << (x < y) << '\n';  // true
  std::cout << (y < x) << '\n';  // false
  std::cout << (x < z) << '\n';  // true
  std::cout << (x == y) << '\n'; // false
}
```


# Assignment operator

### Solution 1
```cpp
class Vector {
public:
	Vector& operator=(const Vector& v) { // Можно писать: a = b = c;
		if (this == &v) { // иначе удалится и то, что копируем
			return this;
		}
		delete[] data;
		data = ...
	}
	...
};
```
### Solution 2 - _Copy and swap_
```cpp
class Vector {
public:
	Vector& operator=(const Vector& v) { // Можно писать: a = b = c;
		Vector copy = v;  // calls copy constructor
		Swap(copy);
	}
	void Swap(Vector& v) { ... }
};
```

# Arithmetical operator
### Solution 1 - внутри класса
```cpp
class Int {
public:
	// ...
	Int operator+(const Int& a) {
		return Int(val + a.val);
	}
private:
	int val{0};
};

int main() {
	s1 + s2; // эквивалентно s1.operator+(s2);
	10 + s1; // CE
}
```

### Solution 2 - снаружи класса
```cpp
Int operator+(const Int& a, const Int& b) { ... }

int main() {
	10 + Int(10); // работает, если конструктор Int не explicit и есть неявное преобразование, т.к. попадает в общий список всех перегрузок
}
```

### Правила хорошего тона для арифметических операторов
- Бинарные операторы определяем вне класса для неявного каста
- Бинарные операторы выражаем через версию с присваиванием (`+` => `+=`)

# Comparison operator
- Лучше делать вне класса
- Вообще не обязательно определять ВСЕ операторы сравнения:
	- через `==` можно выразить `!=`
	- через `<` можно выразить ВСЕ (в т.ч. `==`, однако это 2 вызова `<`)
```cpp
bool operator==(const Int& lhs, const Int& rhs) {
  return !(lhs < rhs) && !(rhs < lhs);
}
```

- `std::rel_ops` творит чудеса
```cpp
#include <iostream>
#include <type_traits>
#include <utility>

struct IntHolder {
  int x{0};
  bool operator==(const IntHolder& other) const {
    return x == other.x;
  }
};

int main() {
  IntHolder a{1}, b{2};
  std::cout << std::boolalpha << '\n';
  using namespace std::rel_ops;
  std::cout << (a == b) << '\n';
  std::cout << (a != b) << '\n';
}
```

```cpp
#include <iostream>
#include <utility>

struct S {
  bool operator==(const S& other) const {
    return true;
  }

  bool operator<(const S& other) const {
    return false;
  }
};


int main() {
  S s, s1;
  using namespace std::rel_ops;

  std::cout << std::boolalpha;
  std::cout << (s == s1) << '\n';
  std::cout << (s < s1) << '\n';
  std::cout << (s > s1) << '\n';
}
```
### `std::less_equal<T>{}`
```cpp
#include <functional>
#include <vector>
#include <iostream>
#include <algorithm>

int main() {
  std::vector v(20, 2);
  std::sort(v.begin(), v.end(), std::less_equal<int>{});  // UB, требования компаратора: cmp(a, a) всегда false
}
```

# Increment & decrement
### Prefix
```cpp
Int& Int::operator++() {
	*this += 1;
	return *this;
}
```

### Suffix
```cpp
Int Int::operator++(int) { // костыль
	Int copy = *this;
	*this += 1;
	return copy;
}
```

# Functor
- Функтор - объект, у которого определен `operator()`, т.е. может вести себя как функция
- Обычно метод константный: `operator()(...) const { ... }`

# `operator[]`
```cpp
class Vector {
	// ...
	int& operator[](int) { return ... }
	const int& operator[](int) const { return ... } // отдельно const-версия
	// ...
};
```

- С C++23 `operator[]` может принимать любое количество аргументов
- В `operator[]` не принято проверять валидность индекса, для этого есть `at()`
- `operator[]` не всегда возвращает `type&`: `vector<bool>::operator[]` возвращает `bit_reference`

# `operator&&` / `operator||`
- Можно перегружать
- Но теряется свойство ленивого вычисления

# `operator*` / `operator->`
### `T& operator*`
- Возращает ссылку на объект, который хотим получить из объекта
### `T* operator->`
- Возвращает то, к чему можно применить стрелку, пока это не дойдет до базового типа

# Оператор ввода / вывода в поток
```cpp
std::ostream& operator<<(std::ostream& os, const S& s) {
	os << s.a;
	return os; // для chain'а: std::cout << a << b << c;
}
std::istream& operator>>(...) { ... }  // аналогично
```

# User-defined literals
Allows integer, floating-point, character, and string literals to produce objects of user-defined type by defining a user-defined suffix.
- Since C++11
- [Source in cppref](https://en.cppreference.com/w/cpp/language/user_literal)
- Also known as suffix operators

### Allowed parameter lists
1. `(const char*)`
2. `(unsigned long long int)`
3. `(long double)`
4. `(char)`
5. `(wchar_t)`
6. `(char8_t)`
7. `(char16_t)`
8. `(char32_t)`
9. `(const char*, std::size_t)`
10. `(const wchar_t*, std::size_t)`
11. `(const char8_t*, std::size_t)`
12. `(const char16_t*, std::size_t)`
13. `(const char32_t*, std::size_t)`

- (1) Literal operators with this parameter list are the _raw literal operators_, used as fallbacks for integer and floating-point user-defined literals
- (2) Literal operators with these parameter lists are the first-choice literal operator for user-defined integer literals
- (3) Literal operators with these parameter lists are the first-choice literal operator for user-defined floating-point literals
- (4-8) Literal operators with these parameter lists are called by user-defined character literals
- (9-13) Literal operators with these parameter lists are called by user-defined string literals
- Default arguments are not allowed

### Example

```cpp
#include <iostream>

double operator""_abs(long double value) {
  return (value > 10 ? -value : value);
}

std::string operator""_to_str(const char* string) {
  return std::string(string);
}

std::size_t operator""_cringe(const char* str, std::size_t len) {
  return len;
}

template <char... Cs>
std::size_t operator""_count() {
  return sizeof...(Cs);
}

int main() {
  std::cout << 13.0_abs << '\n';
  std::cout << 1234_to_str << '\n';
  std::cout << "ksekgchm"_cringe << '\n';
  std::cout << 123456_count << '\n';
}
```

# C-style cast operators
```cpp
#include <iostream>

double x = 10;

struct S {
  explicit operator int() const {
    return 123;
  }

  operator double&() const {
    return x;
  }

  operator const double&() const {
    return x;
  }
};

void foo(int x) {
  std::cout << x << '\n';
}

void bar(double& x) {
  std::cout << x << '\n';
}

int main() {
  S s;
  std::cout << (int)s << '\n';
  std::cout << static_cast<int>(s) << '\n';
  // foo(s);  // CE
  bar(s);
}
```


# Spaceship operator
- Since C++20
- Defined in header `<compare>`

| Type                    | Ordering | Implicitly convertible to                     |
| ----------------------- | -------- | --------------------------------------------- |
| `std::strong_ordering`  | ЛУМ*     | `std::partial_ordering`, `std::weak_ordering` |
| `std::weak_ordering`    | ЛУМ*     | `std::partial_ordering`                       |
| `std::partial_ordering` | ЧУМ**    | -                                             |

- $*$Частично упорядоченное множество
- $**$Линейно упорядоченное множество

### `std::strong_ordering` structure
```cpp
// public static member constants
inline constexpr std::strong_ordering less;
inline constexpr std::strong_ordering equivalent;
inline constexpr std::strong_ordering equal;
inline constexpr std::strong_ordering greater;
```

### `std::weak_ordering` structure
```cpp
// public static member constants
inline constexpr std::weak_ordering less;
inline constexpr std::weak_ordering equivalent;
inline constexpr std::weak_ordering greater;
```

### `std::partial_ordering` structure
```cpp
// public static member constants
inline constexpr std::partial_ordering less;
inline constexpr std::partial_ordering equivalent;
inline constexpr std::partial_ordering greater;
inline constexpr std::partial_ordering unordered;
```

```cpp
#include <iostream>


int main() {
  // <=> returns one of "strong ordering, weak_ordering, partial_ordering"
  int x = 3;
  int y = 10;
  x <=> y;      // strong ordering
  2.5 <=> 3.5;  // partial ordering cause of NaN
                //
  // result of <=> can be compared with other result or with 0
  std::cout << std::boolalpha;
  std::cout << ((x <=> y) < 0) << '\n';
  std::cout << ((y <=> x) > 0) << '\n';
  std::cout << ((x <=> x) == 0) << '\n';
  std::cout << ((x <=> x) == std::strong_ordering::equal) << '\n';
}
```

### Mathematics example
```cpp
#include <ios>
#include <iostream>

struct A {};

struct S {
  int operator<=>(const A&) const {
    return 0;
  }
};

int main() {
  S s;
  A a;
  std::cout << std::boolalpha;
  std::cout << (typeid(s <=> a) == typeid(int)) << '\n';  // true
  std::cout << (typeid(a <=> s) == typeid(int)) << '\n';  // false
  std::cout << (typeid(a <=> s) == typeid(std::strong_ordering)) << '\n';  // (a <=> s)  ==>  (0 <=> (s <=> a))  ==>  (0 <=> int)  ==>  std::strong_ordering ==> true
}
```

### `std::compare_X_order_fallback`
- It's unspecified
- Из `<`, `==` собирает полноценный `std::X_ordering`
	- Словно аналог `using namespace std::rel_ops;`

```cpp
#include <compare>
#include <ios>
#include <iostream>

// Type with support for old-style comparison
struct Coord {
  int x;
  int y;
  friend bool operator<(const Coord& left, const Coord& right) {
    if (left.x == right.x) return left.y < right.y;
    return left.x < right.x;
  }
  friend bool operator==(const Coord& left, const Coord& right) {
    return left.x == right.x && left.y == right.y;
  }
};

int main() {
  std::boolalpha(std::cout);
  Coord a{1, 2}, b{2, 2};

  // Wouldn't compile, type doesn't implement operator<=>
  // auto v = a <=> b;

  // Produces strong_ordering using operator == and <
  auto s = std::compare_strong_order_fallback(a, b);
  // std::is_lt(s) == true
  // decltype(s) == std::strong_ordering

  std::cout << "std::is_lt(s) == " << std::is_lt(s) << '\n';
  static_assert(std::is_same_v<decltype(s), std::strong_ordering>);

  // Produces weak_ordering using operator == and <
  auto w = std::compare_weak_order_fallback(a, b);
  // std::is_lt(w) == true
  // decltype(w) == std::weak_ordering

  std::cout << "std::is_lt(w) == " << std::is_lt(w) << '\n';
  static_assert(std::is_same_v<decltype(w), std::weak_ordering>);

  // Produces partial_ordering using operator== and < (both a < b, b < a)
  auto p = std::compare_partial_order_fallback(a, b);
  // std::is_lt(p) == true
  // decltype(p) == std::partial_ordering

  std::cout << "std::is_lt(p) == " << std::is_lt(p) << '\n';
  static_assert(std::is_same_v<decltype(p), std::partial_ordering>);
}
```

==TODO==
`std::strong(weak/partial)_order and X_order_fallback`
[S1](https://en.cppreference.com/w/cpp/utility/compare/strong_order)
[S2](https://en.cppreference.com/w/cpp/utility/compare/compare_three_way)
[S3](https://gitlab.com/Wanaphi/mipt_cpp_cs_seminars/-/blob/main/c_plus_plus_/02_operator_overloading/desk_photos_/photo_2024-09-21_03-24-12.jpg?ref_type=heads)
