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
	// Это называется "захватом переменной". По умолчанию они константные
	[a](int x) { std::cout << a + x; }(10);  // 20
}
```

- Syntax: `[a, b, c](args){}`
	- `a`, `b`, `c` - константные копии
- `mutable` сделает все захваченные переменные изменяемыми
	- `[a, b, c]() mutable {}`
- By ref: `[&a, b, &c](int x) { ... }`
	- `a`, `c` можно менять, т.е. будет работать как обычная ссылка

# Type of lambda
- Лямбда - класс с `operator() const`
- `sizeof(any lambda) >= 1` - пустой класс с одним методом
	- `[a](){}` - 4 байта (лежит копия `int a`)
	- `[&a](){}` - 8 байт (лежит ссылка на `int`)
- `typeid(lambda).name()`

# Operations with lambdas
### Copy
```cpp
int main() {
  int a = 10;

  auto f = [a](int x, int y) { return x < y; };
  auto ff = f;
}
```
### Move
==TODO==
### Assign
```cpp
int main() {
  int a = 10;

  auto f = [a](int x, int y) { return x < y; };
  auto ff = f;

  ff = f;  // CE
}
```

- С C++20 можно, если список захвата пуст
```cpp
int main() {
  auto f = [](int x, int y) { return x < y; };
  auto ff = f;

  ff = f;
}
```

# Список захвата
### Захват полей класса и this
- Мы можем захватывать ТОЛЬКО локальные переменные
	- То есть глобальные переменные и поля объекта передать в capture-list мы не можем
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [this](int x) { std::cout << a; return x; };
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

#### Захват по ссылке
```cpp
class S {
	int a = 1;
	void foo() {
		auto f = [&a=a])(int x) { std::cout << a; return x; };
	}
};
```

#### Захват по перемещению (Capture by Move)
```cpp
std::string str = "Hello world";

auto lambda = [s = std::move(str)](){ std::cout << s << '\n'; };

lambda();  // Hello world
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

#### Захват всех локальных переменных
- В том числе и `this`
- BAD CODE!
```cpp
struct S {
	int a = 1;
	void foo() {
		auto f = [=](int x) { return a; };
	}
};
```

#### Захват всех локальных переменных по ссылке
- BAD CODE!
```cpp
struct S {
	int a = 1;
	void foo() {
		auto f = [&](int x) { return a; };
	}
};
```

### Example: Context
- Конструкция `[=]` - захват ТОЛЬКО локальных переменных, так называемый local non static context
- В данном примере:
	- `b` честно захватится с конструкцией `[=]`
	- `a` - переменная глобальной области видимости, `offset` до нее (ее данных) вычисляется в compile-time
	- `c` - статическая переменная функции. Хранится в секции `data`, а не на стеке, и `offset` до нее также вычисляется в compile-time
	- `sizeof(f) = 4` - всего 1 `int`
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
- _Closure type_ definition: The lambda expression is a prvalue expression of unique unnamed non-union non-aggregate class type, known as closure type, which is declared (for the purposes of ADL) in the smallest block scope, class scope, or namespace scope that contains the lambda expression. 
- The `+` triggers a conversion to a plain old function pointer for the lambda
	- The compiler sees the first lambda (`[]{}`) and generates a closure object according to §5.1.2 (`[expr.prim.lambda]`). As the lambda is a **non-capturing** lambda
	- This is important as the unary operator `+` has a set of built-in overloads, specifically this one (see `[over.built]`): `T* operator+(T*);`
	- And with this, it's quite clear what happens: When operator `+` is applied to the closure object, the set of overloaded built-in candidates contains a conversion-to-any-pointer and the closure type contains _exactly one candidate_: The conversion to the function pointer of the lambda.
- [Source](https://stackoverflow.com/questions/18889028/a-positive-lambda-what-sorcery-is-this)

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
- In this example, the type of `f1` in `auto f1 = +[]{};` is therefore deduced to `int(*)(int)`. Now the second line is easy: For the second lambda/closure object, an assignment to the function pointer triggers the same conversion as in the first line. Even though the second lambda has a different closure type, the resulting function pointer is, of course, compatible and can be assigned.

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

