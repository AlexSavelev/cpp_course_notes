# Numeric types
## Integers
- Переполнение:
	- Беззнаковое - арифметика в $Z(2^n)$
	- Знаковое - UB
- Integer promotion:
	- `int` -> `long long`
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
### Cast order
1. `const_cast`
2. `static_cast`
3. `static_cast + const_cast`
4. `reinterpret_cast`
5. `reinterpret_cast + const_cast`

# Conversions
- `T` -> `[cv] T[&]`
- `Derived&` -> `Base&`


cv - qualifiers
f(const int&)
f(volatile int&)
f(int)
f(5)

==TODO== cv
==TODO== mutable

[Link](https://gitlab.com/Wanaphi/mipt_cpp_cs_seminars/-/blob/main/c_plus_plus_/01_oop_basics_/06_const_methods.cpp?ref_type=heads)
volatile size_t size;
```cpp
for (size_t i = 0; i < size; ++i) {
 // ...
}

```