# Numeric types
## Integers
- Переполнение:
	- Беззнаковое - арифметика в $Z(2^n)$, где $n$ - битность числа
	- Знаковое - UB
- Integer promotion:
	- `int` -> `long long`
	- `signed` -> `unsigned` - это боль, потому что биты знакового типа начитают интерпретироваться как биты беззнакового
- Size comparison:
	- `char` $<=$ `short` $<=$ `int` $<=$ `long` $<=$ `long long`
- Ranges
	- `int32_t x`: $[$INT32_MIN; INT32_MAX$]$
		- `INT32_MAX = 0x7FFFFFFF`
		- `INT32_MIN = 0x80000000`
		- `INT32_MAX + 1` is **UB**
	- `uint32_t x`: $[$UINT32_MIN; UINT32_MAX$]$
		- `UINT32_MIN = 0x00000000`
		- `UINT32_MAX = 0xFFFFFFFF`
		- `UINT32_MAX + 1 = 0` is **OK**
- Запись чисел в 8- и 16-ти ричных системах счисления
	- Префиксы `0` и `0x`
	- `int a = 012; // 1*2 + 8 * 1 = 10`
	- `int b = 0x12; // 1 * 2 + 16 * 1 = 18`

## Double
- Sign - 1
- Exponent - 11
- Mantissa - 52
- Model: `IEEE 754`
- [More info](https://neerc.ifmo.ru/wiki/index.php?title=%D0%9F%D1%80%D0%B5%D0%B4%D1%81%D1%82%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5_%D1%87%D0%B8%D1%81%D0%B5%D0%BB_%D1%81_%D0%BF%D0%BB%D0%B0%D0%B2%D0%B0%D1%8E%D1%89%D0%B5%D0%B9_%D1%82%D0%BE%D1%87%D0%BA%D0%BE%D0%B9)


# Static variables

```cpp
int foo() {
	static int x = 0; // хранится в секции data (см. Memory)
	std::cout << ++x;
}
```

# Casts
[More info](https://stackoverflow.com/questions/332030/when-should-static-cast-dynamic-cast-const-cast-and-reinterpret-cast-be-used)
## static_cast
- Выполняет преобразования, извесные компилятору
- _**Note**_ `static` он потому что проверяет возможность конвертации на этапе компиляции
```cpp
int a = 5;
double d = static_cast<double>(a);
```

## reinterpret_cast
- Работа с сырой памятью
- Интерпретирует байты памяти одного типа как байты памяти другого типа
- Умеет приводить **только** указатели и ссылки, поскольку не создает новую сущность
```cpp
int a = 0;
double* b = reinterpret_cast<double*>(&a);
```

## const_cast
- Навешивание и снятие константности

#####  Example 1
```cpp
int f(int&) { std::cout << 1; }
int f(const int&) { std::cout << 2; }

main:
	int a = 10;
	f(a); // 1
	f(const_cast<const int&>(a)); // 2
```
##### Example 2
```cpp
int a = 10;
const int& b = a;
int& c = const_cast<int&> b; // OK
```
##### Example 3
```cpp
const int& b = 3;
int& c = const_cast<int&>(b); // очень серьезный UB
```

## C-style cast
- По очереди применяет все касты (поэтому юзать не надо)
- Лучше явно писать нужный каст
- В отличие от остальных C++ кастов, C-style-каст можно перегружать
### Cast order
1. `const_cast`
2. `static_cast`
3. `static_cast + const_cast`
4. `reinterpret_cast`
5. `reinterpret_cast + const_cast`

# Conversions
- `T` -> `[cv] T[&]`
- `Derived&` -> `Base&`

# About CV qualifier
- [Source](https://en.cppreference.com/w/cpp/language/cv)
## Introduction
Any (possibly incomplete) type other than function type or reference type is a type in a group of the following four distinct but related types:
- A _cv-unqualified_ version
- A _const-qualified_ version
- A _volatile-qualified_ version
- A _const-volatile-qualified_ version

## `const` and `volatile` objects
When an object is first created, the cv-qualifiers used (which could be part of decl-specifier-seq or part of a declarator in a declaration, or part of type-id in a new-expression) determine the constness or volatility of the object, as follows:

### A `const` object is
- an object whose type is const-qualified, or
- a non-mutable subobject of a const object.

Such object cannot be modified: attempt to do so directly is a compile-time error, and attempt to do so indirectly (e.g., by modifying the const object through a reference or pointer to non-const type) results in undefined behavior.

### A `volatile` object is
- an object whose type is volatile-qualified,
- a subobject of a volatile object, or
- a mutable subobject of a const-volatile object.

Every access (read or write operation, member function call, etc.) made through a glvalue expression of volatile-qualified type is treated as a visible side-effect for the purposes of optimization (that is, within a single thread of execution, volatile accesses cannot be optimized out or reordered with another visible side effect that is sequenced-before or sequenced-after the volatile access. This makes volatile objects suitable for communication with a signal handler, but not with another thread of execution, see `std::memory_order`). Any attempt to access a volatile object through a glvalue of non-volatile type (e.g. through a reference or pointer to non-volatile type) results in undefined behavior.

- [Article (RU, Habr)](https://habr.com/ru/articles/673428/)

#### Example
- Пишем многопоточную игру, в которой у игрока может быть какое-то количество денег
- Механика сбора денег осуществляется во вторничном потоке, а проверка - в первичном:
```cpp
size_t money = 0;  // изначально денег нет
// ...
// в рамках этого потока не меняем значение money
// ...
if (player_wants_to_buy_a_sword()) {
	if (money > 100) {  // меч стоит 100 монет
		give_sword();
	} else {
		name_player_as_poor();
	}
}
```
- Проблема состоит в том, что компилятор вправе оптимизировать механику и всегда называть игрока бедным
- Решение - навесить `volatile`:
```cpp
volatile size_t money = 0;
```

### A `const volatile` object is
- an object whose type is const-volatile-qualified,
- a non-mutable subobject of a const volatile object,
- a const subobject of a volatile object, or
- a non-mutable volatile subobject of a const object.

Behaves as both a const object and as a volatile object.

Each cv-qualifier (const and volatile) can appear at most once in any cv-qualifier sequence. For example, const const and volatile const volatile are not valid cv-qualifier sequences.

# `mutable` specifier
- `mutable` - permits modification of the class member declared mutable even if the containing object is declared const (i.e., the class member is mutable).

May appear in the declaration of a non-static class members of non-reference non-const type:
```cpp
class X {
    mutable const int* p;  // OK
    mutable int* const q;  // ill-formed
    mutable int&       r;  // ill-formed
};
```

# Conversions
There is partial ordering of cv-qualifiers by the order of increasing restrictions. The type can be said _more_ or _less_ cv-qualified than:

- _unqualified_ < const
- _unqualified_ < volatile
- _unqualified_ < const volatile
- const < const volatile
- volatile < const volatile

References and pointers to cv-qualified types can be implicitly converted to references and pointers to more cv-qualified types.
To convert a reference or a pointer to a cv-qualified type to a reference or pointer to a less cv-qualified type, `const_cast` must be used.

==TODO== https://en.cppreference.com/w/cpp/language/implicit_cast.html#Qualification_conversions

# Full `CV` example
```cpp
#include <cstdlib>
 
int main() {
    int n1 = 0;          // non-const object
    const int n2 = 0;    // const object
    int const n3 = 0;    // const object (same as n2)
    volatile int n4 = 0; // volatile object
 
    const struct {
        int n1;
        mutable int n2;
    } x = {0, 0};        // const object with mutable member
 
    n1 = 1;   // OK: modifiable object
//  n2 = 2;   // error: non-modifiable object
    n4 = 3;   // OK: treated as a side-effect
//  x.n1 = 4; // error: member of a const object is const
    x.n2 = 4; // OK: mutable member of a const object isn't const
 
    const int& r1 = n1; // reference to const bound to non-const object
//  r1 = 2; // error: attempt to modify through reference to const
    const_cast<int&>(r1) = 2; // OK: modifies non-const object n1
 
    const int& r2 = n2; // reference to const bound to const object
//  r2 = 2; // error: attempt to modify through reference to const
//  const_cast<int&>(r2) = 2; // undefined behavior: attempt to modify const object n2
 
    [](...){}(n3, n4, x, r2); // see also: [[maybe_unused]]
}
```
