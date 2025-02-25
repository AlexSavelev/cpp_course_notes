
# CE - compilation error
- Лексические (lexical)
	- `123bad456`
- Синтаксические (semantics)
	- `int + 10`
- Семантические
	- undefined operation
		- `5.0 % 0.5`
	- ambiguous call
	- access error

# RE - runtime error
- segmentation fault - обращение к "плохой" памяти
- div by `0`
- exception

# UB - undefined behavior
- infinite loop
- обращение к элементу за границами массива
- переполнение signed
- *Note*: UB means: it's not a C++ program

# UB - unspecified behavior
```cpp
void f() { std::cout << 1; }
void g() { std::cout << 2; }
void h() { std::cout << 3; }

int main() {
	cout << f() * g() + h(); // порядок вычислений значений аргументов функции
}
```
- Depends on compiler
- Is not documented
- Is not in C++ standard

# Implementation Defined behavior
- C++ code is valid
- Program behavior depends on compiler realization
	- `sizeof(int)`, `sizeof(T*)`
	- STL realization (`std::set<T>` cans be: AVL-tree, Red-black-tree)
	- mapping in `reinterpret_cast` (5.2.10.3) (mapping - отображение)
- It's documented
- [More  about UdfB, UspB, IDB](https://habr.com/ru/articles/450910/)

# Ill-formed
- !well-formed
- no right C++ program

# Ill-formed, nodiagnostic required (IFNDR)
- [More about](https://devblogs.microsoft.com/oldnewthing/20240802-00/?p=110091)

==TODO== UnspB - C++17??
