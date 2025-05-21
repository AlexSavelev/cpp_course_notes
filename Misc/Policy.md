- Политика/стратегия - паттерны программирования

- Простейший пример политики
	- Синтаксические требования политики: наличие статичного метода `handle(int)`
```cpp
#include <iostream>

struct IdPolicy {
  static auto handle(auto x) { return x; }
};

struct SquarePolicy {
  static auto handle(auto x) { return x * x; }
};

template <typename T>
struct CastPolicy {
  static auto handle(auto x) { return static_cast<T>(x); }
};

template <typename Policy = IdPolicy>
auto Foo(auto x) {
  return Policy::handle(x);
}

int main() {
  std::cout << Foo<IdPolicy>(10) << '\n';
  std::cout << Foo<SquarePolicy>(5.5) << '\n';
  std::cout << Foo<CastPolicy<int>>(5.5) << '\n';
}
```
==TODO== add exception policy (Dolta)
# Class policy

==TODO== Dolta