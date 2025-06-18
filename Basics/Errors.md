
# CE - compilation error
- Лексические (lexical)
	- `123bad456`
- Синтаксические (syntax)
	- `int + 10`
	- missed `;`
- Семантические (semantics)
	- undefined operation
		- `5.0 % 0.5`
	- ambiguous call
	- access error
	- `a + b = c;`
- Linked error
	- Undefined reference to `main`

# RE - runtime error
- segmentation fault - обращение к "плохой" памяти
- div by `0`
- uncaught exception
- double free (is not exception)

# UB - undefined behavior
- `[defns.undefined]` behavior for which this document imposes no requirements
- infinite loop
- обращение к элементу за границами массива
- переполнение signed
- *Note*: UB means: it's not a C++ program
- [See more](https://en.cppreference.com/w/cpp/language/ub)

```cpp
int main() {
  // При компиляции с -O2 получается вечный цикл
  for (int i = 0; i < 300; ++i) {
    std::cout << i << ' ' << i * 12345678 << std::endl;
  }
}
```

# UB - unspecified behavior
- `[defns.unspecified]` behavior, for a well-formed program construct and correct data, that depends on the implementation
- В отличии от implementation defined, разработчики систем/компиляторов не обязаны в документации это поведение описывать
- The behavior of the program varies between implementations, and the conforming implementation is not required to document the effects of each behavior.
```cpp
#include <iostream>

int f() { std::cout << 1; return 0; }
int g() { std::cout << 2; return 0; }
int h() { std::cout << 3; return 0; }

int main() {
  std::cout << f() * g() + h();
  // порядок вычислений значений аргументов функции
}
```
- Examples
	- Evaluation order
	- Whether identical [string literals](https://en.cppreference.com/w/cpp/language/string_literal "cpp/language/string literal") are distinct
		- The compiler is allowed, but not required, to combine storage for equal or overlapping string literals. That means that identical string literals may or may not compare equal when compared by pointer.
		- `[lex.string]` Whether all string literals are distinct (that is, are stored in nonoverlapping objects) and whether successive evaluations of a string-literal yield the same or a different object is unspecified.
- Depends on compiler
- Is not documented
- Is not in C++ standard

# Implementation Defined behavior
- `[defns.impl.defined]` behavior, for a well-formed program construct and correct data, that depends on the implementation and that each implementation documents
- C++ code is valid
- Program behavior depends on compiler realization
	- `sizeof(int)`, `sizeof(T*)`, `sizeof(size_t)`
	- number of bits in a byte
	- text of `std::bad_alloc::what`
	- STL realization (`std::set<T>` cans be: AVL-tree, Red-black-tree)
	- mapping in `reinterpret_cast` (mapping - отображение)
- It's documented
- [More  about UdfB, UspB, IDB](https://habr.com/ru/articles/450910/)

# Ill-formed
- `[defns.ill.formed]` ill-formed program is a program that is not well-formed
	- `[defns.well.formed]` C++ program constructed according to the syntax rules, diagnosable semantic rules, and the one-definition rule
- no right C++ program

```cpp
const int size = 4;

void expand() {
    // ill-formed: Modifying a const variable.
    size = 9;
}
```
- The above program is ill-formed because it tries to modify a const variable, and the compiler reports a compile-time error and refuses to produce a program. The plain vanilla ill-formed programs aren’t scary because the compiler lets you know that you broke a rule and typically refuses to let you proceed until you fix it.

# Ill-formed, nodiagnostic required (IFNDR)
- The program has semantic errors which may not be diagnosable in general case (e.g. violations of the ODR or other errors that are only detectable at link time).
- These are programs which are ill-formed, but for which the standard does not require the compiler to report an error. The compiler is welcome to do so if it chooses, but it is also permitted to remain silent. In practice, IFNDR is used to describe things which are “bad” but which compilers are not equipped to detect.
- For example, if you have two translation units (standard-speak for “a .cpp file”), both of them must agree on the bodies of any inline functions.
```cpp
// file1.cpp

inline int magic() { return 42; }
int get_value() { return magic(); }

// file2.cpp

extern int get_value();
inline int magic() { return 99; }

int main(int argc, char** argv) {
    if (argc > 1) return get_value();
    return 0;
}
```
- In the above example, we have a project that consists of two .cpp files. The two files disagree on what the `int magic()` inline function does, which is a category of IFNDR. The compiler is permitted but not required to detect this mismatch, and if you run the resulting program, the results are undefined: If you run the resulting program with a command line argument, the `get_value()` function might return 42. It might return 99. It might return 31415. It might reformat your hard drive. It might hang.
- Even worse, even if you run the program with no command line options (so that `get_value()` is never called), the results are _still_ undefined. It could still reformat your hard drive.
- [Source](https://devblogs.microsoft.com/oldnewthing/20240802-00/?p=110091)

# SFINAE
- Substitution Failure Is Not An Error
- [About](https://en.cppreference.com/w/cpp/language/sfinae)
- This rule applies during overload resolution of function templates: When substituting the explicitly specified or deduced type for the template parameter fails, the specialization is discarded from the overload set instead of causing a compile error
- This feature is used in template metaprogramming
- Also see in Metaprogramming page
