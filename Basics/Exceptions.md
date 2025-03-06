# Example
```cpp
#include <iostream>

void foo(int x, int y) {
  if (y == 0) {
    throw std::string("Wrong Number\n");
  }
  if ( x / y == 3) {
    std::cout << "Success\n";
  }
}

int main() {
  int x;
  foo(6, 2);
  foo(10, 0);
  foo(9, 3);
}
```

- Бросить можно все, что душе угодно
```cpp
#include <iostream>

// Бросить можно что угодно

int foo() {
  throw std::abort;  // function
}

int main() {
  try {
    foo();  
  } catch ( decltype(std::abort) f ) {
    std::cout << "Catched\n";
    f();
  }
}
```

# Stack unwinding
- При броске выделяется блок памяти, туда укладывается объект, который вы бросили, начинает разворачиваться стек (stack unwinding)

# Try-catch

```cpp
#include <iostream>

void foo(int x, int y) {
  if (y == 0) {
    throw std::string("Wrong Number\n");
  }
  if (x / y == 3) {
    std::cout << "Success\n";
  }
}

int bar() {
  try {
    foo(10, 0);
  } catch (int x) {
    std::cout << x << '\n'; 
  }
  return 0;
}

int main() {
  int x = 0;
  try {
    bar();
  } catch (std::string s) {
    std::cout << s << '\n';
  }

  std::cout << "Reached End\n";
  std::cout << x << '\n';
}
```

### Multi
```cpp
#include <iostream>

// В "динамической" памяти создается буффер, помещается объект исключения и начинает разворачиваться стек
// На эту память можно взять ссылку

int main() {
  try {
    throw 1;  // не должно отвечать за control flow
  } catch (int& x) {
    std::cout << x << '\n';
  }
  
  // possible conversion: add/remove const volatile &
  try {
    throw 3ll;
    // throw std::abort;
  } catch (std::string) {
    std::cout << "string\n";
  } catch (double) {
    std::cout << "double\n";
  } catch (long long) {
    std::cout << "long long\n";
  } catch (const long long) {
    std::cout << "const long long\n";
  } catch (...) /* поймает что угодно */ {
    std::cout << "caught something\n";
  }
}
```

### Copy on throw
```cpp
#include "Verbose.hpp"

int main() {
  try {
    Verbose v;  // Verbose 1 default constructed
    throw v;  // Verbose 2 move constructed
    // Verbose 1 destructed
  } catch (Verbose v) {  // Verbose 3 copy constructed
	// Verbose 3 destructed
	// Verbose 2 destructed
  }

  Verbose v1;  // Verbose 4 default constructed

  try {
    Verbose v;  // Verbose 5 default constructed
    throw v;  // Verbose 6 move constructed
    // Verbose 5 destructed
  } catch (Verbose& v) {
    // Verbose 6 destructed
  }

  try {
    throw Verbose{};  // Verbose 7 default constructed
  } catch (Verbose&) {
	// Verbose 7 destructed
  }

  throw Verbose{};  // Verbose 8 default constructed
}
```
==TODO== Verbose 4 destructor not called?
### Conversions
```cpp
#include <iostream>

struct Base {};

struct Derived : Base {};

int main() {
  // objects: add/remove const volatile &
  // pointers: many
  try {
    throw "Hello\n";  // const char[7]
  } catch (const char*) {
    std::cout << "Catched\n";  // OK
  }

  Derived d;
  try {
    throw &d;
  } catch (Base*) {
    std::cout << "D catched\n";  // OK
  }

  // objects: cast ref from derived to base
  try {
    throw Derived{};
  } catch (Base& b) {
    std::cout << "D& Catched\n";  // OK
  }
}
```

### Conversions 2
```cpp
#include <iostream>

struct A {};

struct B : A {};
struct C : A {};

struct D : B, C {};

struct AV {};

struct BV : virtual AV {};
struct CV : virtual AV {};

struct DV : BV, CV {};

int main() {
  D d;
  // A& aref = d;
  try {
    throw D{};
  } catch (A& ) {
    std::cout << "catched\n";
  } catch (...) {
    std::cout << "...\n";
  }

  try {
    throw DV{};
  } catch (AV&) {
    std::cout << "Caught\n";
  }

}
```

### Rethrow
- Исключения можно обрабатывать, а после опрокидывать на следующий уровень с помощью `throw;`
- Причем если исключение не было опрокинуто сверху, то конструкция `throw;` кидает исключение, которые не получится поймать (т.е. сразу `abort`)
- Пример ниже выводит: Hello Catched
```cpp
#include <exception>
#include <iostream>
#include <stdexcept>

void bar() {
  throw std::runtime_error{"Ayaya"};
}

void foo() {
  try {
    bar();
  } catch (std::exception& ex) {
    std::cout << "Hello\n";
    throw;
  }
}

int main() {
  try {
    foo();
  } catch (std::runtime_error& err) {
    std::cout << "Catched\n";
  } catch (std::exception& err) {
    std::cout << "Exc\n";
  }

  try {
    throw;  // uncaught
  } catch (...) {
    std::cout << "here\n";
  }
}
```

# Гарантии безопасности
- Нет
	- Ничего не гарантируем
- Слабая
	- Данные не полетят и можно будет продолжать с ними работать
- Сильная
	- Состояние объекта не изменилось

# Terminate
- Можно задать свои инструкции при `terminate`
```cpp
#include <exception>
#include <iostream>

void foo() {
  std::cerr << "My Terminate called\n";
}

int main() {
  std::set_terminate(foo);

  // std::terminate();
  throw 1;
}
```

# `noexcept`
```cpp
#include <exception>
#include <ios>
#include <iostream>

void foo() {
  throw 1;
}

void bar() noexcept;

void baz() noexcept(sizeof(int) > 4) {

}

void FOO() noexcept(noexcept(bar())) {}

void BAR() {}

int main() {
  std::cout << std::boolalpha;
  std::cout << noexcept(foo()) << '\n';  // false
  std::cout << noexcept(5 * 6) << '\n';  // true
  std::cout << noexcept(3 / 0) << '\n';  // true
  std::cout << noexcept(bar()) << '\n';  // true
}
```

### `noexcept` throw
```cpp
#include <iostream>

void foo() noexcept {
  try {
    throw 1;
  } catch (int x) {
    std::cout << x << '\n';
  }

  throw 3;  // std::terminate();
}

int main() {
  try {
    foo();
  } catch (...) {
    std::cout << "catched\n";  // NO
  }
}
```

### Use of `noexcept`
- Деструкторы
- Перемещающие операции не ошибаются!
	- Конструкторы перемещения
	- Присваивание по перемещению
