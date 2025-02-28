
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
- exception
- double free (is not exception)

# UB - undefined behavior
- `[defns.undefined]` behavior for which this document imposes no requirements
- infinite loop
- обращение к элементу за границами массива
- переполнение signed
- *Note*: UB means: it's not a C++ program
- [See more](https://en.cppreference.com/w/cpp/language/ub)

# UB - unspecified behavior
- `[defns.unspecified]` behavior, for a well-formed program construct and correct data, that depends on the implementation
- The behavior of the program varies between implementations, and the conforming implementation is not required to document the effects of each behavior.
```cpp
void f() { std::cout << 1; }
void g() { std::cout << 2; }
void h() { std::cout << 3; }

int main() {
	cout << f() * g() + h(); // порядок вычислений значений аргументов функции
}
```
- Examples
	- Evaluation order
	- Whether identical [string literals](https://en.cppreference.com/w/cpp/language/string_literal "cpp/language/string literal") are distinct ==TODO==
- Depends on compiler
- Is not documented
- Is not in C++ standard

# Implementation Defined behavior
- `[defns.impl.defined]` behavior, for a well-formed program construct and correct data, that depends on the implementation and that each implementation documents
- C++ code is valid
- Program behavior depends on compiler realization
	- `sizeof(int)`, `sizeof(T*)`, `sizeof(size_t)`
	- number of bits in a byte
	- text of std::bad_alloc::what
	- STL realization (`std::set<T>` cans be: AVL-tree, Red-black-tree)
	- mapping in `reinterpret_cast` (5.2.10.3) (mapping - отображение)
- It's documented
- [More  about UdfB, UspB, IDB](https://habr.com/ru/articles/450910/)

# Ill-formed
- `[defns.ill.formed]` ill-formed program is a program that is not well-formed
	- `[defns.well.formed]` C++ program constructed according to the syntax rules, diagnosable semantic rules, and the one-definition rule
- no right C++ program

# Ill-formed, nodiagnostic required (IFNDR)
- The program has semantic errors which may not be diagnosable in general case (e.g. violations of the ODR or other errors that are only detectable at link time).
- [More about](https://devblogs.microsoft.com/oldnewthing/20240802-00/?p=110091)  ==TODO==

