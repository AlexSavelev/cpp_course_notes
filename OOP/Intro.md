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
- `sizeof(any instance of class) >= 1` всегда (даже если классы stateless, т.е. без полей)

# `operator.`, `operator->`
- `.` - оператор для работы с объектами
- `->` - оператор для работы с указателями на объект
- `this` - указатель на текущий объект (писать его не надо)
# Constructors
- `T second = first; // вызывается не operator=, а конструктор`

### `explicit`
- `explicit` - запрет этого неявного каста (`Vector a = 10;` не равносильно `Vector(size: 10);`)
```cpp
struct A {
  A(int a, int b = 10) {}
};

int main() {
  A a = 10;  // OK

  return 0;
}
```

- Также запрещает также неявный каст при вызове `operator Type()`
```cpp
struct Demo {
    explicit operator bool() const { return true; }
};

int main() {
	demo d;
	if (d);        // OK, вызывает Demo::operator bool()
	bool b_d = d;  // ОШИБКА: не может преобразовать 'Demo' в 'bool' во время инициализации
	bool b_d = static_cast<bool>(d);    // OK, явное преобразование, вы знаете, что делаете
}
```
- Может принимать `constexpr`-выражение: `explicit(1 == 2)`
### Initializer list
- Будет проинициализировано в порядке указания полей в самом классе
- `Student(...) : name_(name), grade_(grade) { ... }`

### Делегирующий конструктор
- Since C++11
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
- Генерируется компилятором всегда (пустой)
- Не надо самостоятельно вызывать деструктор (иначе UB)
	- Можно только в том случае, когда дважды он точно не вызовется
- Syntax: `~Student() { ... }`

# Порядок вызова
## При создании
1. Конструкторы полей класса
2. Конструктор класса

## При удалении
1. Деструктор класса
2. Деструктор полей класса

# Friend
- Отношение "A - это друг B" не симметрично и не транзитивно
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
	- Это, к слову, используется в Loophole'ах

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
==TODO== friend ADL [SOF](https://stackoverflow.com/questions/23831077/why-does-friend-function-found-successfully-via-adl) and [GL](https://gitlab.com/yaishenka/cpp_course/-/blob/main/sems/03_libs_and_ADL/05_friend_ADL.cpp?ref_type=heads) and [GL2](https://gitlab.com/yaishenka/cpp_course/-/blob/main/sems/03_libs_and_ADL/06_cringe.cpp?ref_type=heads) and [GL3](https://gitlab.com/Wanaphi/mipt_cpp_cs_seminars/-/blob/main/c_plus_plus_/02_operator_overloading/07_friend_ADL.cpp?ref_type=heads)

# Static class members
- Статичные поля/методы являются являются общими для всего класса

```cpp
struct S { static int x; };

int main() {
	S::x;  // = 0
}
```

- _**Note:**_ Можно вызывать статические методы из экземпляра таким же образом, как и из класса.
	- Это позволяет изменять нестатические методы экземпляра на статические без необходимости обновления записи вызова функции.
```cpp
struct Foo {
  static void Bar() {}
};

int main() {
  Foo f;
  f.Bar();  // Вызовы эквивалентны
  Foo::Bar();
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

# Access modifiers
- Проверка на доступность осуществляется после выбора перегрузки

```cpp
#include <iostream>

struct S {
  void foo(int) { std::cout << "int\n"; }

 private:
  void foo(bool) { std::cout << "double\n"; }
};

int main() {
  S s;
  s.foo(3);
  s.foo(true);  // CE
}
```

# Правило трёх
- Если класс такой, что вы определили нетривиальный конструктор, деструктор или оператор присваивания, то скорее всего, вам нужно реализовать все их.
- Правило трех это исключительно словесное правило, компилятор ему не следует
