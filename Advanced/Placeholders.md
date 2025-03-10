A placeholder type specifier designates a _placeholder type_ that will be replaced later, typically by deduction from an initializer.

# `auto`
- Позволяет иногда не писать тип создаваемого объекта
- `for(auto it = map.begin(); it != map.end(); ++it) { ... }`
- `for(auto item : map) { ... }`

- Type is deduced using the rules for template argument deduction
```cpp
int x;
auto y = x;  // int
```

- Можно навешивать ссылки и константность
```cpp
const auto& y = x;
```

- Ссылки вообще всегда обрезаются
- `auto&&` работает как универсальная ссылка

### `auto` в качестве возвращаемого значения
- Очень полезно, когда тип большой или его нельзя выразить явно
	- Например, фабрика лямбда функций
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
- Честно берется `int`
```cpp
auto f(int x) -> int {
	return x;
}
```

### `auto` в аргументах функций
- since C++20
- Буквально сокращение шаблонов
```cpp
void f(auto&& x) {  // сокращение шаблонов

}
```
- Можно с вариадиками
```cpp
void f(auto... x) {

}
```

### `auto` как шаблонный аргумент
- See `auto` templates in Templates page

# `decltype`
- Метафункция, которая в compile-time возвращает тип выражения
```cpp
int x;
decltype(x) u = x;
```

Вывод типов в `decltype(expr)` работает по следующим правилам:
a) if the value category of expression is xvalue, then `decltype` yields `T&&`;
b) if the value category of expression is lvalue, then `decltype` yields `T&`;
c) if the value category of expression is prvalue, then `decltype` yields `T`.
То есть `decltype` возвращает точный тип (с сохранением ссылок)

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
- Это происходит потому что `const` навешивается не на то что лежит под ссылкой а на саму ссылку. А ссылки и так нельзя перевешивать (т.е. они и так константные)

### Example 2
- Код внутри `decltype` не выполняется - это compile-time конструкция
	- Он просто из сигнатуры выражения возьмет возвращаемый тип
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
decltype(some_func(x)) g(const T& x) {
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
- Механизм выводить тип не по правилам `auto` а по правилам `decltype`

```cpp
template <typename Container>
auto ...
```

- `decltype(auto)` можно так же использовать в других выражениях (при инициализации например)
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

==TODO== https://devblogs.microsoft.com/oldnewthing/20201015-00/?p=104369
