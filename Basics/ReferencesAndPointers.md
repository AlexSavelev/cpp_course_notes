# Pointers
- Взаимодействуем не с физической ОЗУ, а с виртуальной - той, что ОС выделила под прогу

### `operator&`
- Унарный оператор
- Возращает адрес объекта (`T` -> `T*`)

### `operator*`
- Унарный оператор
- По адресу возращает значение (разыменование)
- `int value = *ptr; // Разыменование`

### Операции с указателями
- `T* ptr;`
- `++` / `--` - сдвигает на `sizeof(T)` байт
- `+n` / `-n` - сдвигает на `n * sizeof(T)` байт

- `void*` - указатель на что угодно (сырая память)
- `nullptr` - нулевой указатель

# References
- Модификатор типа (можно сказать, новый тип (`int&`))
	- К слову, ссылки появились лишь в C++ (в C их нет)
- Хотим создавать псевдонимы (синонимы) для переменных
- Хотим легковесный способ передачи аргументов (без указателей)

```cpp
int a = 1;
int& b = a;
int c = 1;
b = c; // <=> a = c // ссылку нельзя отвязать
b++;
std::cout << a; // 2
```

- Нельзя перегружать функции по ссылке:
	- `foo(int&)`, `foo(int)` => CE
- Ссылка принимает **только** l-value
	- `int&* a = &c; // CE`
- _**Note**_ ссылка привязывается на все время своего существование. То есть если у нас есть ссылка `second` которая указывает на переменную `first`, то эту ссылку нельзя перепривязать ни к чему другому.

# Works with constants

### Pointer to constant
```cpp
int a = 0;
int b = 1;
const int* ptr = &a; // сам указатель менять можно, а что под ним - нельзя

std::cout << *ptr; // 0
*ptr = 10; // CE
ptr = &b; // OK
```

## Constant pointer

```cpp
int a = 0;
int b = 1;
int* const ptr = &a; // указатель менять нельзя, а что под ним - можно

*ptr = 10; // OK
ptr = &b; // CE
```

## Constant pointer to constant

```cpp
const int* const ptr = &a;
```

- _**Note:**_ Запрещено снимать степень константности, иначе UB

## Pointer example
```cpp
int main() {
  int a = 5;
  const int* p = &a;

  ++p; // OK
  ++*p; // CE

  int* const pp = &a;

  ++pp; // CE
  ++*pp; // OK

  const int* const ppp = pp; // OK (cast from int* to const int*)
  ++ppp; // CE
  ++*ppp; // CE

  int* const q = ppp; // CE (запрещенный каст)
  const int* qq = ppp; // OK, copy
}
```

## Summary
```cpp
char * const str1; // str1 cannot be modified, but the character pointed to can
const char * str2; // str2 can be modified, but the character pointed to cannot
const char * const str3 // neither str3 nor the character pointed to can be modified.
```

# Продление жизни

```cpp
const int& c = 5; // r-value, поменять нельзя => продление жизни временному объекту "5"

void print(const vector<int>& a) { ... } // так писать правильно
```

# Overloading by const/non-const reference

_**Note**_ Можно делать перегрузку по константной и не константной ссылкам.

```cpp
#include <iostream>

void foo(int& a) { std::cout << 1; }
void foo(const int& a) { std::cout << 2; }

int main() {
	int a = 0;
	foo(a); // 1

	int& b = a;
	foo(b); // 1

	const int& c = 5;
	foo(c); // 2
}
```

# `ptrdiff_t`
- Implementation defined
```cpp
// valid since C++11
using ptrdiff_t = decltype(static_cast<int*>(nullptr) - static_cast<int*>(nullptr));
```

