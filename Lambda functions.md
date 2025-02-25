# Syntax
- `[](Arg1 arg1, Arg2 arg2, ...) { ...; return value; }`
	- `[]` - список захвата
	- `()` - список аргументов
	- `{}` - тело лямбда-функции
- `auto f = [](int x, int y) { return x + y }`
	- `f(1, 2);`
- `auto GetFunction() { return [](int x, int y) { return x + y; } };`
	- Например, для фабрики компараторов
- Call in place: `[](int x, int y){ return x + y; }(10, 10)`
- Явное указание возвращаемого типа: `auto f = [](int x, int y) -> int { return x + y; }`
	- Такое себе

# Списки захвата
### Example - CE
```cpp
int main() {
	int a = 10;
	[](int x) { std::cout << a + x; }(10);  // CE
}
```

### Example  - OK
```cpp
int main() {
	int a = 10;
	[a](int x) { std::cout << a + x; }(10);  // 20
}
```

- Syntax: `[a, b, c](args){}`
	- `a`, `b`, `c` - константные копии
- Mutable
	- `[a, b, c]() mutable {}`
	- Можно менять
- By ref: `[&a, b, &c](int x) { ... }`
	- `a`, `c` можно менять и будет работать как обычная ссылка

# Type of lambda
- Лямбда - класс с `operator() const`
- `sizeof(lambda) == 1` - пустой класс с одним методом
	- `[a](){}` - 4 байта (лежит копия `int a`)
	- `[&a](){}` - 8 байт (лежит ссылка на `int`)
- `typeid(lambda).name()`

# Can
- Copy: `auto f = ...; auto ff = f;`
	- CE if f contains local-fields: `[a]`
- Move: coming soon...==TODO==

# Список захвата для класса
- Верно и для функций
### Example
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [this])(int x) { std::cout << a; return x; };
		// we can't write [a], because a is not local variable (we can't enter non-local variables)
	}
};
```

### Захват с инициализацией (since C++14)
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [b = a + 10])(int x) { std::cout << b; return x; };
	}
};
```

#### Захват по ссылке:
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [&a=a])(int x) { std::cout << a; return x; };
	}
};
```

#### Захват по константной ссылке
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [&a=std::as_const(a)])(int x) { std::cout << a; return x; };
	}
};
```

#### Захват всех локальных переменных - BAD CODE
```cpp
struct S {
	int a = 1;
	void foo() {
		auto f = [=](int x) { return a; };
	}
};
```

#### Захват всех локальных переменных по ссылке - BAD CODE
```cpp
struct S {
	int a = 1;
	void foo() {
		auto f = [&](int x) { return a; };
	}
};
```

### Example: Context
==TODO== cppref
```cpp
#include <iostream>

int a = 0;


auto foo() {
  int b = 1;
  static int c = 2;
  ++c;
  // local non static context
  return [=](int d){
    return a + b + c + d;
  };
}

int main() {
  auto f = foo();
  std::cout << f(10) << '\n';
  a = 10;
  std::cout << f(10) << '\n';
  auto f2 = foo();
  std::cout << f(10) << '\n';
  std::cout << f2(10) << '\n';
}
```

# Positive lambda hack
- [About](https://stackoverflow.com/questions/18889028/a-positive-lambda-what-sorcery-is-this)
- ==TODO==
### Example
```cpp
#include <iostream>

// lambda closures with empty capture list has convertion to pointer to function

int main() {
  auto f1 = +[](int x){ return x * 2; };  // positive lambda hack
  auto f2 = +[](int x){ return x * x; };
  f1 = f2;
  
  auto f3 = []{ std::cout << "Hello\n"; };
  f3();
}
```

# Template lambdas
```cpp
#include <iostream>
#include <vector>

int main() {
  auto f = [](auto value) { return value * 2; };
  auto f1 = []<typename T>( const std::vector<T>& vec ) -> std::size_t { return vec.size(); };

  std::vector<int> v{1, 2, 3, 4};
  std::cout << f1(v) << '\n';
}

```
- `auto` работает как `template`

```cpp
#include <iostream>

int main() {
  auto f = [](auto value) { std::cout << typeid(value).name() << '\n'; };

  f(1);
  f(2.0);
}
```

# Lambda overload
==TODO== think about
```cpp
#include <iostream>
#include <variant>
#include <vector>

template <typename... Args>
struct Overloaded : Args... {
  using Args::operator()...;
  Overloaded(Args... args) {}
};

template <typename... Args>
Overloaded(Args... args) -> Overloaded<Args...>;

int main() {
  std::variant<int, std::string, std::vector<int>> var = 10;
  
  auto visitor = Overloaded(
      [](auto x) { std::cout << "sizeof int = " << sizeof(x) << '\n'; },
      [](std::string s) { std::cout << s << '\n'; },
      []<typename T>(std::vector<T>) { std::cout << "called for vector\n"; }
  );
  std::visit(visitor, var);

  var = "Hello";
  std::visit(visitor, var);

  var = std::vector<int>{1, 2, 3, 4};
  std::visit(visitor, var);
}
```

