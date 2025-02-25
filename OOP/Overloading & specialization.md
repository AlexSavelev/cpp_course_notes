# Overloading of template functions
Основные замечания:
- Частное лучше общего
- Если есть более частная версия, но нужно будет сделать приведение типов, то лучше более общая версия
#### Example 1
```cpp
template <typename T>
void f(T x) { std::cout << 1; }

void f(int x) { std::cout << 2; }  // overloading

int main() {
	f(0); // 2 - более частный случай
}
```

#### Example 2
```cpp
template <typename T, typename U>
void f(T x, U y) { std::cout << 1; }

template <typename T>
void f(T x, T y) { std::cout << 2; }

void f(int x, double y) { std::cout << 3; }

int main() {
	f(0, 0); // 2 (2 лучше 1, т.к. более частная, 2 лучше 3, т.к. не надо делать приведение типов)
}
```

#### Example 3
```cpp
template <typename T>
void f(T x) { std::cout << 1; }

template <typename T>
void f(T& x) { std::cout << 2; }

int main() {
    int x = 0;
    int& y = x;
    f(y); // CE - ambiguous call (см. ссылки)
}
```

#### Example 4
```cpp
template <typename T>
void f(T& x) { std::cout << 1; }

template <typename T>
void f(const T& x) { std::cout << 2; }

int main() {
    int x = 0;
    int& y = x;
    f(y); // 1 - константность еще надо навесить
}
```

# Template specialization
==TODO== https://en.cppreference.com/w/cpp/language/template_specialization
```cpp
template <typename T>
struct Vector {
	void foo() { ... }
};

template <>
struct Vector<bool> { // Полная специализация шаблонного класса (придется заного написать класс, он не будет иметь ничего общего с Vector<T>)
	std::string g(std::vector<int>&) { ... }
};
```

```cpp
#include <iostream>

template <typename T>
T SumOfTwo(const T& lhs, const T& rhs) { return lhs + rhs; }

template int SumOfTwo(const int& lhs, const int& rhs); // явная инстанциация

template <>
int SumOfTwo(const int& lhs, const int& rhs); // объявление специализации - CE (specialization of ‘T SumOfTwo(const T&, const T&) [with T = int]’ after instantiation)

int SumOfTwo(const int& lhs, const int& rhs); // объявление перегрузки

template <typename T>
void bar() {
  SumOfTwo<T>(2, 3);
}

int main() {
//   SumOfTwo<int>(3, 2);  
//   SumOfTwo<long long>(3, 2);  
//   SumOfTwo<double>(3, 2);
//   bar<int>();
}
```

# Partial specialization
- Частичная специализация

### Syntax
```cpp
template <typename T>
class Vector<T*> { // Это специализация будет работать для всех T, которые являются указателями
    ...
}
```

```cpp
template <typename T, typename U>
class C {
    ...
}

template <typename T>
class C<T, int> { // работает по тем же правилам, что и перегрузка
    ...
}
```

### Example 1
```cpp
template <typename T, typename U>
struct S {};

template <typename T>
struct S <int, T> {};

template <typename T, typename... Args>
struct S <T, std::tuple<Args...>> {};
```

### Example 2
```cpp
#include <iostream>

template <typename T, typename U>
struct FooImpl {
  static void call(T, U) {
    std::cout << 1 << '\n';
  }
};

template <typename T, typename U>
struct FooImpl<T, const U> {
  static void call(T, const U) {
    std::cout << 2 << '\n';
  }
};

template <typename T, typename U>
void foo(T& t, U& u) {
  return FooImpl<T, U>::call(t, u);
}

int main() {
  int x;
  const int y = 0;
  foo(x, x);  // 1
  foo(x, y);  // 2
}

```
# Examples

### Example 1
- _**Note:**_ Вторая функция является уже перегрузочной, поскольку с `f<T, U>` ничего общего не имеет
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

template <typename T>
void f(T, T) { std::cout << 2; }

template <>
void f(int, int) { 3 }

int main() {
	f(0, 0); // 3 (среди перегрузок мы выбрали вторую, так как она более частная. А у нее уже выбрали специализацию)
}
```

### Example 2
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

template <>
void f(int, int) { std::cout << 3; }

template <typename T>
void f(T, T) { std::cout << 2; }

int main() {
	f(0, 0); // 2 (Специализация применилась к 1 функции. На моменте выбора перегрузки выиграла 2, так как она более частная. У 2 версии нет специализации, поэтому она вызвалась сама
}
```

### Example 3
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

// тут template убрали
void f(int, int) { std::cout << 3; }

template <typename T>
void f(T, T) { std::cout << 2; }

int main() {
	f(0, 0); // 3 (потому что теперь это тоже перегрузка)
}
```

### Example 4
```cpp
#include <iostream>

template <typename T>
void foo(T) {
  std::cout << 1 << '\n';
}

template <typename T>
void foo(T*) {

  std::cout << 2 << '\n';
}

template <>
void foo(int) {
  std::cout << 3 << '\n';
}

int main() {
  foo(10);  // 3
}

```

### Example 5
```cpp
#include <iostream>

template <typename T, typename U>
void bar(T, U) {
  std::cout << 1 << '\n';
}

template <typename T, typename U>
void bar(T*, U*) {
  std::cout << 2 << '\n';
}


template <>
void bar<int*, int*>(int*, int*) {
  std::cout << 3 << '\n';
}  // Эта функция - специализация первой функции (не второй), Если убрать <int*, int*>, но выберет под основу то вторую. Тест на внимательность

int main() {
  int* ptr = nullptr;
  bar(ptr, ptr);  // 2
}
```

### Example 6
```cpp
#include <iostream>

template <typename T, typename U>
void baz(T, U) {
  std::cout << 1 << '\n';
}

template <>
void baz(int*, int*) {
  std::cout << 2 << '\n';
}

template <typename T, typename U>
void baz(T*, U*) {
  std::cout << 3 << '\n';
}

template <>
void baz(int*, int*) {
  std::cout << 4 << '\n';
}

int main() {
  int* ptr = nullptr;
  baz(ptr, ptr);
  baz<int*, int*>(ptr, ptr);  // 4 - выберет специализацию (4) наиболее подходящего шаблона (3)
}
```

### Example 7
```cpp
#include <iostream>

template <typename T>
void foo(T) {
  std::cout << "T\n";
}

struct S {};

template <typename T>
void call(T t, S s) {
  foo(t);
  foo(s);
}

void foo(S) {
  std::cout << "S\n";
}

int main() {
  S s;
  call(s, s);  // ST
  // s - независимо от шаблона, для foo(s) существует только шаблон
  // t - зависимо от шаблона, зарезолвить foo(t) получится уже после того, как найдется более подходящая перегрузка
}
```

==TODO== fixup examples