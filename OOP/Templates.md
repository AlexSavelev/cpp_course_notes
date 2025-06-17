# Introduction
- Процесс подстановки шаблонов называется "инстанцирование шаблонов", происходит он на этапе транслирования (см. Compiling)
- Если никогда не вызывалась, то в итоговом программном коде ее не будет (поэтому definition пишется в header'е, т.к. должна компилироваться с вызывающем коде)
### Example

```cpp
template <typename T>
T GetMax(const T& lhs, const T& rhs) {
	return lhs > rhs ? lhs : rhs;
}

int main() {
	int test = GetMax<int>(1, 2);
}
```

### Cringe example
```cpp
template <typename T, typename U>
const T& GetMax(const T& a, const U& b) {
	return a > b ? a : b;
}

GetMax(1, 2.0); // UB - возвращаю константную ссылку на временный объект (double -> int)
```

### Aliases
- Алиасы и псевдонимы - синонимы
- Шаблонные алиасы появились в C++11
```cpp
template <typename T>
using MyMap = std::map<T, T>

MyMap<int> a; // std::map<int, int> a;
```

### Template constants
- It's cringe
```cpp
template <typename T>
const T pi = 3.14; // Cringe

pi<int>, pi<double>, ...
```

### Template method of template class
```cpp
template <typename T>
struct Vector {
	// Declaration
	template <typename U>
	void push_back(const U&);
};

// Definition
template <typename T>
template <typename U>
void Vector<T>::push_back(const U& a) { ... }
```

# Правила вывода шаблонных типов

### С помощью `= delete`
```cpp
template <typename T>
void foo(T x) = delete;

foo(1); // CE with type output: [with T = int]
```
#### Example 1
```cpp
template <typename T>
void foo(T x) = delete;

int x = 0;
int& y = 0;
const int& z = y;
const int c = 0;

f(x); // int
f(y); // int
f(z); // int
f(c); // int
```

#### Example 2
```cpp
template <typename T>
void f(T& x) = delete;

int x = 0;
int& y = x;
const int& z = y;
const int c = 0;

f(x); // int
f(y); // int
f(z); // const int
f(c); // const int
```

#### Example 3
```cpp
template <typename T>
void f(const T& x) = delete;

int x = 0;
int& y = x;
const int& z = y;
const int c = 0;

f(x); // int
f(y); // int
f(z); // int
f(c); // int
```

#### Example 4
- Ссылки на ссылку не существует, поэтому в следующем случае выведется просто `int&`
```cpp
template <typename T>
void f(T& x) = delete;

int x = 1;
int& y = x;

f<int&>(y); // int& & -> int&
```

### Pretty function
- Позволяет показать информацию о вызываемой уже инстанцированной функции
```cpp
template <typename T>
void foo() {
  std::cout << __PRETTY_FUNCTION__ << '\n';
}

int main() {
  foo<int>();
  foo<std::string>();
}
```

# Template parameters
- Шаблонные параметры, как правило, пишутся стилем `CamelCase`
	- `typename T`, `bool IsConst`, `size_t BucketSize`, `class Allocator`
	- `typename` и `class` - это эквивалетные вещи, но пишем `typename`
### Non-type template parameters
#### Numeric parameters
```cpp
template <typename T, int N>
struct Array {
	T arr_[N]; // создается на стеке, т.к. N - compile-time const
};

Array<int, 10> a1;
Array<int, 20> a2;
a1 = a2; // CE - different data types
```

- _**Note**_ Шаблонными параметры, не являющимися `typename`, `class` могут быть только целочисленные типы а также `bool` и `char` (неверно после C++20)

```cpp
int d = 0;
std::cin >> d;
std::array<int, d> arr1; // CE - non-const

int c = 10;
std::array<int, c> arr1; // CE даже если не меняем ее - определено стандартом

const int y = c;
std::array<int, y> arr1; // CE - константа от неконстанты (runtime const)

const int x = 10;
std::array<int, x> arr1; // OK (compile-time const)
```

#### `auto` parameters
- See `auto` templates in Templates page

### Шаблонные параметры шаблонов
- Например, если пишем свой `Stack` и ходим передавать в шаблонах, какой контейнер использовать в качестве адаптера

```cpp
template <typename T, template <typename> typename Container>
class Stack {
	public:
	...
	private:
	Container<T> container_;
};

int main() {
	Stack<int, std::vector> stack;
}
```

### Шаблонные аргументы по умолчанию
```cpp
template <typename T, typename Cmp = std::less<T>>
void sort(...) { ... }
```

# Overloading and specialization
- см. [[OverloadingAndSpecialization]]

# Проблема зависимых имен

### Преамбула
```cpp
template <typename T>
struct S {
	using X = T;
};

template <>
struct S<int> {
	static int X;
};

int a = 0;

template <typename T>
void f() {
	S<T>::X * a;
}
```

- Что это по итогу?
- При `[T != int]`: declaration - взяли `X` из `S` и объявили указатель, который назвали `a`
- При `[T = int]`: expression - взяли `X` из `S` и умножили на глобальную `a`
- _**Def**_ Собственно, это и есть "проблема зависимых имен", здесь `X` зависит от `T`.
### Solution v1
- Есть следующее правило: если компилятор встречает зависимое имя и не может понять что это, то он по умолчанию считает что expression, поэтому если вызвать только `f()` то все будет ок
- Если нужно, чтобы компилятор парсил зависимое имя как declaration, то нужно написать `typename` (это вторая роль слова `typename` в C++)
```cpp
template <typename T>
void f() {
    typename S<T>::X * a;
}
```

### Solution v2
- Рассмотрим еще пример:
```cpp
template <typename T>
struct S {
    template <int M, int N>
    struct A{};
}

template <>
struct S<int> {
    static const int A = 0;
}

template <typename T>
void f() { S<T>::A<1,2> a; }
```
- Строку можно понять как объявление переменной a типа `S::A<1,2>`
- А также как `S::A` $<$ `1` , `2` $>$ `a`
	- Операторы сравнения + оператор запятая

- Даже если добавим `typename`, это не поможет, потому что слово `typename` лишь означает что дальше будет название типа, а не шаблон
- Чтобы это пофиксить нужно переписать функцию `f` вот так:
```cpp
template <typename T>
void f() {
    typename S<T>::template A<1,2> a;
}
```

# Basic type traits
- Шаблоны позволяют создавать метафункции (функции от типов)
- Все нижеперечисленные метафункции реализованы в STL и находятся в заголовке `type_traits`
### `is_same`
```cpp
template <typename U, typename V>
struct is_same {
	static const bool value = false;
};

template <typename U>
struct is_same<U, U> { // specialization
	static const bool value = true;
};

// using
template <typename U, typename V>
void f(U x, V y) {
    if (is_same<U,V>::value) {
        // ...
    }
}
```

### `remove_reference`
```cpp
template <typename T>
struct remove_reference {
	using type = T;
};

template <typename T>
struct remove_reference<T&> {
	using type = T;
};

template <typename T>
struct remove_reference<T&&> {  // see move semantics
	using type = T;
};
```

### `remove_const`
- `CE` example
```cpp
template <typename T>
void foo(T x) {
    T y;
}

int main() {
	foo<const int>(x); // CE
}
```

- Если в качестве шаблонного параметра указан константный тип и внутри функции необходимо создать переменную данного типа но уже без константности, то можно написать метафункцию `remove_const`:

```cpp
template <typename T>
struct remove_const {
    using type = T;
};

template <typename T>
struct remove_const<const T> {
    using type = T;
};

template <typename T>
void foo(T x) {
     typename remove_const <T>::type y;
}
```

### `std::decay`
- Разлагает тип
```BrainFuck
Let U be remove_reference_t<T>. If is_array_v<U> is true, the
member typedef type equals remove_extent_t<U>*. If
is_function_v<U> is true, the member typedef type equals
add_pointer_t<U>. Otherwise the member typedef type equals
remove_cv_t<U>.
```

```cpp
template<class T>
struct decay {
 private:
  typedef typename std::remove_reference<T>::type U;
 public:
  typedef typename std::conditional<std::is_array<U>::value,
	  typename std::add_pointer<typename std::remove_extent<U>::type>::type,
	  typename std::conditional<std::is_function<U>::value,
            typename std::add_pointer<U>::type,
            typename std::remove_cv<U>::type
      >::type
  >::type type;
};
```

### С алиасами
- Начиная с C++11 существуют шаблонные псевдонимы, поэтому всеми этими метафункциями стало чуть проще пользоваться:
	- `using remove_const_t = typename remove_const<T>::type`
- В С++17 добавили шаблонные константы, поэтому стало еще лучше:
	- `template <typename U, typename V> const bool is_same_v = is_same<U, V>::value;`

# Variadic templates & fold expressions
- Появился в C++11
- Шаблон с переменным числом параметров
- `args` называется пакетом

```cpp
void print() {}

template <typename Head, typename... Tail>
void print(const Head& head, const Tail&... tail) {
	std::cout << head << '\n';
	print(tail...);
}
```

### Вариадики помогают писать `type-traits`
```cpp
template <typename First, typename Second, typename... Types>
struct is_same_many { // is_homogeneous
    const static bool value = std::is_same_v<First, Second>::value && is_same_many<Second, Types...>::value;
};

template <typename First, typename Second>
struct is_same_many<First, Second> { // is_homogeneous
    const static bool value = std::is_same_v<First, Second>;
};
```

### Example
```cpp
#include <iostream>

int f(int x) { return x; } 
int f(int x, int y) { return x + y; } 
int f(int x, int y, int z) { return x + y + z; } 
int f(int x, int y, int z, int t) { return x + y + z + t; }

template <typename... Args>
int foo(const Args&... args) {
  return f(f(args...) + f(args)...);
  // f(f(1, 2, 3) + f(args)...);
  // f(f(1, 2, 3) + f(1), f(1, 2, 3) + f(2), f(1, 2, 3) + f(3));
  // f(6 + 1, 6 + 2, 6 + 3); = 24
}

template <typename... Args>
int bar(const Args&... args) {
  return f(f(args, args...)...);
  // f(f(args, 1, 2, 3)...);
  // f(f(1, 1, 2, 3), f(2, 1, 2, 3), f(3, 1, 2, 3))
  // f(7, 8, 9) = 24
}

int main() {
  std::cout << foo(1, 2, 3) << '\n';
  std::cout << bar(1, 2, 3) << '\n';
}

```

### Unpacking and features
```cpp
print((tail + 1)...) // +1 к каждому аргументу
sizeof...(tail) // sizeof пакета
```

- `(<pack> <operator> ...)`
	- Example: `sum(1, 2, 3, 4, 5) => (args + ...)`
		- $1 + (2 + (3 + (4 + 5)))$
	- Example: `foo(1, 2, 3) => (args, ...)`
		- `operator,` returns $3$ as last element of the pack
	- Example: `bar(1, 2, 3) => ((args * args) + ...)`
		- $1 * 1 + (2 * 2 + (3 * 3 + (4 * 4 + (5 * 5))))$
- `(... <operator> <pack>)`
	- Example: `sum(1, 2, 3, 4, 5) => (... + args)`
		- $(((1 + 2) + 3) + 4) + 5$
- `(<init> <operator> ... <operator> <args>)`
	- Example: `f(1, 2, 3, 4, 5) => (10 + ... + args)`
		- $((((10 + 1) + 2) + 3) + 4) + 5$
	- Example: `Print(const Args&... args): (std::cout << ... << args);`
### Fold expressions
- [See more](https://en.cppreference.com/w/cpp/language/fold)

```cpp
#include <iostream>
#include <format>

template <typename... Args>
int sum(const Args&... args) {
  return (... + (args * args));
}

template <typename... Args>
void Print(const Args&... args) {
  (std::cout << ... << std::format("{} ", args));
}

int main() {
  std::cout << sum(1, 2, 3, 4, 5) << '\n';
  Print(1, 2, 3, 4, 5);
  std::cout << sum() << '\n';
}
```

# Вычисления на шаблонах
- Иногда бывает нужно заставить компилятор посчитать что-то на этапе компиляции
- Такие вычисления называются "метапрограммированием"
- На экзе будет билет на подсчет простоты числа за корень в compile-time

### Example - Fib
```cpp
template <int N>
struct Fib {
    static const int value = Fib<N-1>::value + Fib<N-2>::value;
};

template <>
struct Fib<1> {
    static const int value = 1;
};

template <>
struct Fib<0> {
    static const int value = 0;
};
```

### Example - Prime
```cpp
template <int N, int K>
struct Helper {
    static const bool value = (N % K != 0) && Helper<N, K - 1>::value;
};

template <int N>
struct Helper<N, 1> {
    static const bool value = true;
};

template <int N>
struct IsPrime {
    static const bool value = Helper<N, N-1>::value;
};
```

### recursive template instantiation exceeded maximum depth
- `IsPrime<10000000>::value;
- Компилятор упадет с ошибкой "recursive template instantiation exceeded maximum depth" (по умолчанию 900)
- Чтобы повысить глубину шаблонной рекурсии нужно запустить компиляцию с флагом `-ftemplate-depth=N`
- Глубина рекурсии все равно не может быть слишком большой: оперативки компилятору не хватит

```cpp
template <std::size_t I>
void foo() {
  foo<I + 1>();
}

int main() {
  foo<0>();
}
```

# Template compiling
- Рассмотрим модель пошаговой компиляции C++ кода с шаблонными параметрами на примере приведенного ниже содержимого.
```cpp
template <typename T>
void foo(T) {}

template <typename T>
void foo(T&) {}

template <typename T>
void foo(T&, std::vector<T>&) {}

void foo(int) {}

int main() {
	foo(3);
}
```

1. Name lookup (смотрит на все, что подходит по имени) (не смотрит на сигнатуру)
2. Отсеить неподходящие (количество аргументов)
3. Вывод типов (template deduction) (в рамках вызова `foo(3.14)` взять `[T = double]`)
4. Подстановка (template substitution) (только в сигнатуру функции (не тело))
	- Создается overload set, и то, что substitution failed, выбрасывается из общего списка. Здесь как раз работает SFINAE.
5. Overload resolution (выбор между `foo(T=int)` и `foo(int)`: выберет перегрузку, т.к. туда не надо было выводить тип)
6. Template instantiation (инстанциация) (если в функции тоже используются шаблонные функции, компилятор будет рекурсивно начинать процесс заного - уже для этой шаблонной функции)
	- `template <typename T> void bar(T) {}`
	- `template <typename T> void foo(T t) { bar(t) }`

#### Example on `f("Hello")` invoke

| Args                          | State                                      |
| ----------------------------- | ------------------------------------------ |
| `int`                         | Overload resolution failed ==TODO== cppref |
| `double`                      | Overload resolution failed                 |
| `T`                           | OK - `T = [char*]`                         |
| `std::type_identity<T>`       | deduction failed                           |
| `std::vector<T>`              | deduction failed                           |
| `T` (and returns `T::size_t`) | substitution failed                        |
| `int, int`                    | wrong number of args                       |

- Если нет шаблонов: name lookup, overload resolution only ==TODO==

# Argument deduction
- [More info](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction)
- Строка `std::vector v1(10, 2);` равносильна `std::vector<int> v(10, 2);` ввиду того, что компилятор может определить тип автоматически, на основе переданных в конструктор параметров.

```cpp
#include <vector>

template <typename T>
void foo(std::vector<T>) {

}

template <typename T>
void bar(std::vector<T>::value_type) {} // non-deduced context - по using (aliases, алиасам) нельзя понять

int main() {
	std::vector<int> v;
	foo(v);
	bar(v);  // CE
}
```

- Однако вывод типов не работает для аргументов по умолчанию
```cpp
template <typename T>  // or =int
void g(T&& value = 10) {}

int main() {
	g();  // CE (вывод типов не работает для аргументов по умолчанию)
	g<int>();  // OK
	g(11);  // OK
}
```

### Type identity
- From `<type_traits>`
- Solution for disable argument deduction
```cpp
template <typename T>
void foo(std::type_identity_t<T>) {}

int main() {
	foo(10); // CE
	foo<int>(10); // OK
}
```

```cpp
template <typename T>
struct TypeIdentity {
	using Type = T;
}

template <typename T>
using TypeIdentityT = TypeIdentity<T>::Type;
```

### Deduction guide

```cpp
#include <iostream>

template <typename T>
struct S {
	template <typename U>
	S(U) {}
	
	S() {}
	
	S(int) {}
};

template <typename T>
S(T) -> S<T>;

S() -> S<int>;

S(int) -> S<void>;

template <typename T>
void Print(T) {
	std::cout << __PRETTY_FUNCTION__ << '\n';
}

int main() {
	S s(3.5);
	Print(s) // S<double>;

	S s1(5);
	Print(s1) // S<void>;

	S s2;
	Print(s2) // S<int>;
}
```

# Friend injection
```cpp
#include <iostream>
#include <type_traits>

// #define T int

template <typename T>
struct Rational {
  Rational() = default;
  Rational(const T& num, const T& denum)
    : numenator_(num), denumenator_(denum) {}

  Rational(const T& val) : numenator_(val), denumenator_(1) {}

  friend Rational<T> operator*(const Rational<T>&, const Rational<T>&) {
    std::cout << "Called operator *\n";
    return {};
  }

  Rational<T> operator/(const Rational<T>&) {
    std::cout << "Called operator /\n";
    return {};
  }
 private:
  T numenator_{0};
  T denumenator_{0};
};

template <typename T>
Rational<T> operator-(const Rational<T>&, const Rational<T>&) {
  std::cout << "Called operator-\n";
  return {};
}


template <typename T>
Rational<T> operator+(const Rational<T>&, const Rational<std::type_identity_t<T>>&) {
  std::cout << "Called operator+\n";
  return {};
}

int main() {
  Rational<int> r1, r2;
  r1 - r2;
  // r1 - 3; -> CE

  r1 + r2;
  r1 + 3;
  // 3 + r1; -> CE
  
  r1 * r2;
  r1 * 3;
  3 * r1;

  r1 / r2;
  r1 / 3;
  // 3 / r1; -> CE

}
```

# `auto` templates
- Since C++17
- [Source](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0127r1.html)
An `auto` keyword in a template parameter can be used to indicate a non-type parameter the type of which is deduced at the point of instantiation. It helps to think of this as a more convenient way of writing:

```cpp
template <typename Type, Type value>
```

For example,

```cpp
template <typename Type, Type value> constexpr Type constant = value;
constexpr auto const IntConstant42 = constant<int, 42>;
```

can now be written as

```cpp
template <auto value> constexpr auto constant = value;
constexpr auto const IntConstant42 = constant<42>;
```

### With variadic template parameters
```cpp
template <auto ... vs> struct HeterogenousValueList {};
using MyList1 = HeterogenousValueList<42, 'X', 13u>;

template <auto v0, decltype(v0) ... vs> struct HomogenousValueList {};
using MyList2 = HomogenousValueList<1, 2, 3>;
```

### Extracting the value of `SIZE` without knowing its type
```cpp
#include <iostream>

template <std::size_t SIZE>
class Foo {};

template <template <auto> class T, auto K>
auto extractSize(const T<K>&) {
  return K;
}

int main() {
  Foo<6> f1;
  Foo<13> f2;
  std::cout << extractSize(f1) << std::endl;  // 6
  std::cout << extractSize(f2) << std::endl;  // 13
}

```
