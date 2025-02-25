- Объектно ориентированное программирование
- Абстракция - выделение значимой информации и исключение из рассмотрения незначимой
- Инкапсуляция - св-во системы объединить данные и методы
- Наследование - св-во системы, позволяющее описать новый класс на основе уже существующих
- Полиморфизм - св-во системы, позволяющее использовать объекты с одинаковым интерфейсом
	- Полиморфизм - это способность функции обрабатывать данные разных типов. (За одним интерфейсом скрываются разные реализации)

# Structures in C
- Умели только сторить поля
```c
struct A {
	int a;
	char b;
	double c;
};

A a;
a.a = 10;
a.b = 'c';
a.c = 1.1; 
```

# Classes & structures in C++
- `class` - кастомный data type, состоящий из набора полей и методов для взаимодействия с ними
- `sizeof(class B) >= 1` всегда (даже если классы stateless, т.е. без полей)

# `operator.`, `operator->`
- `.` - оператор для работы с объектами
- `->` - оператор для работы с указателями на объект
- `this` - указатель на текущий объект (писать его не надо)

# Constructors
- `T second = first; // вызывается не operator=, а конструктор`
- `explicit` - запрет неявного каста (`Vector a = 10;` не равносильно `Vector(size: 10);`)

### Initializer list
- Будет проинициализировано в порядке указания полей в самом классе
- `Student(...) : name_(name), grade_(grade) { ... }`

### Делигирующий конструктор
`S(int a) { ... }`
`S(int a, int b) : S(a) { ... } // но нельзя продолжать список инициализации`

### Генерация методов
- `S(const S&) = delete; // запретили генерировать метод копирования`
- `S() = default; // запросили сгенерировать метод`

### Агрегантная инициализация структур
- Доступна **только** в структурах
- `S s{1, 2, 3, 4, "aaa"};`

# Destructor
- Не принимает аргументов, поскольку вызывается, когда выходим из scope'а
- Не надо самостоятельно вызывать деструктор (иначе UB)
	- Можно только в том случае, когда дважды он точно не вызовется
- `~Student() { ... }`

# Порядок вызова при создании объекта
1. Конструкторы полей класса
2. Конструктор класса
3. Деструктор класса
4. Деструктор полей класса

# Friend
- Не рефлексивна и не транзитивна
- Показатель плохого проектирования класса
```cpp
#include <iostream>

// struct S;
// void bar();

class C {
  private:
    int foo(int x) { return x * x; }

  friend struct S;
  friend void bar();
};

struct S {
  void foo() {
    C c;
    std::cout << c.foo(10) << '\n';
  }
};

void bar() {
  C c;
  std::cout << c.foo(100) << '\n';
}

int main() {
  S s;
  s.foo();
  bar();
}
```
- Получается, конструкция `friend` так же является объявлением функции/класса.

# ADL
- ADL - Argument Dependent Lookup
- Argument-dependent lookup (ADL), is the set of rules for looking up the unqualified function names in function-call expressions, including implicit function calls to overloaded operators. [Source](https://en.cppreference.com/w/cpp/language/adl)
- Simple def: collect names and choose a good one

```cpp
std::vector<int> a = {0, 1};
swap(a[0], a[1]);  // std::swap
```

```cpp
#include <iostream>

namespace A {
  struct S {};
  
  void foo(S) {
    std::cout << "foo called\n";
  }
}

int main() {
  A::S s;
  foo(s);
}
```


==TODO== ADL source rasshar
==TODO== friend ADL [SOF](https://stackoverflow.com/questions/23831077/why-does-friend-function-found-successfully-via-adl) and [GL](https://gitlab.com/yaishenka/cpp_course/-/blob/main/sems/03_libs_and_ADL/05_friend_ADL.cpp?ref_type=heads) and [GL2](https://gitlab.com/yaishenka/cpp_course/-/blob/main/sems/03_libs_and_ADL/06_cringe.cpp?ref_type=heads)

# Static class members

```cpp
struct S { static int x; };

int main() {
	S::x;  // = 0
}
```
### Singleton implementation
- Singleton - паттерн (антипаттерн - godobject - singleton, многое себе позволяющий)
```cpp
struct Singleton {
	static Singleton* Create() {
		if (obj_ == nullptr) {
			obj_ = new Singleton();
		}
		return obj_;
	}

	Singleton(const Singleton&) = delete;

private:
	Singleton() { ... } // private
	static Singleton* obj_;
};
```
