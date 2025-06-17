- _**Def**_ Объявление - введение какой-то новой сущности.
- **Note** Каждое определение является объявлением.

Объявление (declaration) переменной информирует компилятор о том, что где-то, возможно, в другой единице трансляции (очень грубо, в другом cpp-файле) выделено `sizeof` байт под хранение переменной такого-то типа с таким-то именем. Деклараций можно писать сколько угодно в разных блоках кода, по одной на блок.

Определение (definition) переменной информирует тот же компилятор о том, что память под переменную нужно взять прямо в этом месте, где написано данное определение. Именно в этой единице трансляции. Определение на всю программу может быть одно и только одно.
# Объявление
```cpp
double f(int a);

int main() {
	f(1); // CE
}
```

# Определение
```cpp
double f(int a);

double f(int b) {
	return 3.14;
}

int main() {
	f(1); // OK
}
```

# Functions
Declaration: `return_type function_name(ArgType1 arg1, ArgType2 arg2);`
Definition: `return_type function_name(ArgType1 arg1, ArgType2 arg2) { my_code }`
### Aspects
- Аргументы по умолчанию указываются в объявлении
- Можно перегружать
	- При этом в перегрузках можно менять возвращаемое значение - компиляции все равно это не помешает
	- `void foo(double);`
	- `void foo(int);`
	- [Full list](https://en.cppreference.com/w/cpp/language/overload_resolution)
	- В C, кстати, вообще перегрузок нет
- Но есть ambiguous call
	- `void foo(float);`
	- `void foo(int);`
	- `main: foo(5.0); // CE - double -> float OR double -> int ?!`
- Указатель на функцию
	- `return_type(*name)(arg1, arg2) = &f;`
- Переменное число аргументов
	- `cstdarg`
	- `int add_nums(int count, ...) { ... }`

### Перегрузки

Кратко про выбор перегрузки:
- Точное совпадение всегда побеждает
- Если точного совпадения нет, то выигрывает promotion (integer и float)
- Если нет promotion то дальше идет Conversion
```cpp
void foo(float) {}
void foo(int) {}

int main() { foo(5.0); // CE call to foo is ambiguous }
```
_**Note**_ `double` -> `float` и `double` -> `int` это стандартные преобразования равнозначные с точки зрения компилятора

### Default arguments in general
- The default arguments for functions are not bound to the function itself, but to the **calling context**: defaults declared for the function in the scope in which it is called will be used (See C++ standard, `[dcl.fct.default]`) 

```cpp
#include <iostream>

void f(int a = 1);  // forward declaration with a default value for the
// compilation unit

void f(int a) {  // definition
  std::cout << a << '\n';
}

void g() {
  void f(int a = 2);  // declaration in the scope of g
  f();
}

int main() {
  f();  // uses general default => 1
  g();  // uses default defined in g => 2
  return 0;
}
```

- С конструкциями типа:
```cpp
struct S {
  friend void f(int a = 3);
  static void h() { f(); }
};
```
- или добавлением еще одного объявления будет CE

- Example from `[dcl.fct.default]`
```cpp
void g(int = 0, ...);  // OK, ellipsis is not a parameter so it can follow a
                       // parameter with a default argument

void f(int, int);
void f(int, int = 7);

void h() {
  f(3);                  // OK, calls f(3, 7)
  void f(int = 1, int);  // error: does not use default from surrounding scope
}

void m() {
  void f(int, int);      // has no defaults
  f(4);                  // error: wrong number of arguments
  void f(int, int = 5);  // OK
  f(4);                  // OK, calls f(4, 5);
  void f(int, int = 5);  // error: cannot redefine, even to same value
}
void n() {
  f(6);  // OK, calls f(6, 7)
}
template <class... T>
struct C {
  void f(int n = 0, T...);
};
C<int> c;  // OK, instantiates declaration void C::f(int n = 0, int)
```

# ODR - one definition rule

- Simple: Every object has only one definition!
- Every definition is located in `.cpp` file
- If definition in .hpp - write keyword `inline`
- У любой `не-inline` функции или `не-inline ODR-used` объекта должен быть 1 definition на программу
- У классов должен быть 1 definition на 1 единицу трансляции

## `inline` keyword

- in C: copy-paste func-body in code
	- В C++ это тоже может служить подсказкой компилятору, что можно встроить код функции в момент вызова. Однако компилятор может это проигнорировать.
- in C++: allow more than one definitions, I swear there is the same body
	- `inline` - this function will be defined in multiple translation units, don't worry about it. The linker needs to make sure all translation units use a single instance of the variable/function.

#### inline namespace
- allow call function in 1-level up
```cpp
namespace Impl {
	namespace v1 {
	  int foo(int x) {
	    return x + 2;
	  }
	}
	
	inline namespace v2 {
	  int foo(int x) {
	    return x * 2;
	  }
	}
	
	namespace v3 {
	  int foo(int x) {
	    return x * x;
	  }
	}
}

int main() {
  std::cout << Impl::v1::foo(10) << '\n';
  std::cout << Impl::v2::foo(10) << '\n';
  std::cout << Impl::v3::foo(10) << '\n';
  std::cout << Impl::foo(10) << '\n';
}
```

# Function delete
```cpp
void foo(int) = delete;
void foo(double) {}

int main() {
	foo(3.5);
	// foo(1);  // CE
}
```
