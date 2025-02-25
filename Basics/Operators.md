- Arithmetical (16): `+ - * / %` (and `+= -= *= /= %=`) (and `un. +, un. -`) (and `++pref, post++, --pref, post--`)
- Bit (11): `& | ^ ~ << >>` (and `&= |= ^= <<= >>=`)
- Logical (3): `&& || !` (not recommend, forgive lazy)
- Specific (9): `, * ->` (has to return object that has `operator->` too) `->* () []` (from C++23 it's no binary operator) `= C-style_cast &`
	- operator `,`: Matches left arg, right and returns right
	- operator `()`: Invoke functor's method `operator()`
- Compare: (still live in C++17) `== != (!(==)) < > (a < b => b < a) <= (!(b < a)) >= (!(a < b))`
	- `using namespace std::rel_ops;` (from `utility`) allow not write `a > b` if there is `a < b`
	- from C++20 it's default
	- from C++20 also: `<=>` - operator spaceship (return: `std::strong_ordering` (always 99%) (less, greater, equal, equivalent), `std::weak_ordering` (less, greater, equivalent), `std::partial_ordering` (less, greater, equivalent, not_ordered)) (ex: (`3 <=> 5`) == `std::strong_ordering::less`) (objects can compare with 0 or other objects: less < 0, greater > 0, equivalent == equal == 0, not_ordered != 0)
- Can't overload (4): `. .* ?: ::`
	- Can't overload operators of base types (`int`, `ptr`, ...)
	- Can't create new operators (exception: literal suffix operator)

`std::cout << x = 5; // 5`
- С C++17 правая часть operator= вычисляется первой (`++x = x++`)

# Priorities
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
### Solution 2
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
- Делаем вне класса
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
- Also known as suffix operators
- Since C++11
- [cppref](https://en.cppreference.com/w/cpp/language/user_literal)
==TODO== cppref check

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
