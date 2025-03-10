
- _**Def**_ Наследование — концепция объектно-ориентированного программирования, согласно которой абстрактный тип данных может наследовать данные и функциональность некоторого существующего типа, способствуя повторному использованию компонентов программного обеспечения
# Basics
### Example 1
```cpp
struct Base {
	int x = 1;
};

struct Derived : Base { // default = public
	int x = 2;
};

int main() {
	Derived d;
	std::cout << d.x; // 2
	std::cout << d.Base::x; // 1
	std::cout << sizeof(d); // 8 (int + int)
}
```

### Example 2
```cpp
struct Base {
	void foo() { std::cout << 1; }
};

struct Derived : Base {
	void foo(int) { std::cout << 2; }
};

int main() {
	Derived d;
	std::cout << d.foo(); // CE - Derived затмил Base::foo()
	// solution: Base: protected/public foo; Derived: using Base::foo;
}
```

### Private, public, protected inheritance
```cpp
struct Base {};
struct Derived : public Base {};
```

|                        | public inheritance | protected inheritance | private inheritance |
| ---------------------- | ------------------ | --------------------- | ------------------- |
| public base members    | public             | protected             | private             |
| protected base members | protected          | protected             | private             |
| private base members   | not accessible     | not accessible        | not accessible      |

Check the example:
```cpp
class Base {
 public:
  int publicMember;
 protected:
  int protectedMember;
private:
  int privateMember;
};
```

- Everything that is aware of `Base` is also aware that `Base` contains `publicMember`.
- Only the children (and their children) are aware that `Base` contains `protectedMember`.
- No one but `Base` is aware of `privateMember`.

"Is aware of" means "acknowledge the existence of, and thus be able to access".

The same happens with public, private and protected inheritance. Let's consider a class `Base` and a class `Child` that inherits from `Base`.
- If the inheritance is `public`, everything that is aware of `Base` and `Child` is also aware that `Child` inherits from `Base`.
- If the inheritance is `protected`, only `Child`, and its children, are aware that they inherit from `Base`.
- If the inheritance is `private`, no one other than `Child` is aware of the inheritance.

Иначе говоря, про приватное наследование знает только наследник
- [Source](https://stackoverflow.com/questions/860339/what-is-the-difference-between-public-private-and-protected-inheritance)
# Расположение объектов в памяти при наследовании
```cpp
class Base {
  int a;
  char b;
};

class Derived: Base {
  double c;
};
```

- `sizeof(Derived)` = 16:
	- 4 - `int`
	- 1 - `char`
	- 3 - `alignment`
	- 8 - `double`
- Если бы в объекте `Base` не было полей, то `Derived` состоял бы просто из одного `double` (empty base optimization)

### Выравнивание (alignment)
- _**Def**_ Машинном словом называется единица данных, которая выбрана естественной для данной архитектуры
- Процессор считывает из оперативной памяти данные и кладет их в свою память: регистры. Размер регистра это и есть машинное слово
- Соответственно за раз из памяти читается машинное слово: в случае `x86-64` это 8 байт.

```
[байт 0] [байт 1] [байт 2] [байт 3] [байт 4] [байт 5] [байт 6] [байт 7]
         [    нужно это машинное слово     ]
[     но приходится читать это    ] [              и это              ]
```

- Чтобы таких ситуаций не происходило C++ "выравнивает" данные за счет добавления "мусорных" байтов. Выравнивание происходит по 4 байтам
#### Example
```cpp
struct S {
    char x; // 1 байт
    int y;  // 4 байта
    char z; // 1 байт
};
```

```
[  ] [  ] [  ] [  ] [  ] [  ] [  ] [  ] [  ] [  ] [  ] [  ]
[x ] { потерянные } [        y        ] [z ] { потерянные }
```

# Constructors and destructors in inheritance
### Порядок вызова конструкторов и деструкторов
```cpp
#include <iostream>

struct A {
	A() {std::cout << "A\n"; }
	~A() {std::cout << "~A\n"; }
};

struct B {
	B() {std::cout << "B\n"; }
	~B() {std::cout << "~B\n"; }
};

struct Base {
	Base() {std::cout << "Base\n"; }
	~Base() {std::cout << "~Base\n"; }
	A a;
};

struct Derived: Base {
	Derived() {std::cout << "Derived\n"; }
	~Derived() {std::cout << "~Derived\n"; }
	B b;
};

int main() {
	Derived d;
	// A Base B Derived ~Derived ~B ~Base ~A
}
```

### Parent constructor + initializer list
```cpp
struct Base {
	Base(int a) {}
};

struct Derived: Base {
	Derived(int a, double d): Base(a), d_(d) {}
	double d_;
};
```

# Casting to parent/child classes
### Приведение типов
```cpp
struct Base {
	int x;
};

struct Derived : Base {
	int y;
}

int main() {
	Derived d;
	Base b = d; // Base, который лежит в Derived
	Base&& bb = d; // ссылка на Base, где на самом деле лежит Derived
}
```

### Срезка при копировании
`memory of derived: x,y (Base,y)`
`Base* b = &d`
- Происходит следующая проблема: что если Base поддерживает метод, который нелогично применять к Derived.
- Пример: многоугольник поддерживает метод, который сдвигает одну из точек в произвольное место. Прямоугольник такой метод поддерживать не может.
### SOLID
_**Def**_ Принцип подстановки Барбары Лисков: пусть q(x) является свойством, верным относительно объектов x некоторого типа T. Тогда q(y) также должно быть верным для объектов y типа S, где S является подтипом типа T.

_**Note**_ Принцип подстановки Барбары Лисков входит в принципы SOLID:
1. SRP (single responsibility principle) - для каждого класса должно быть определено единственное назначение. Все ресурсы, необходимые для его осуществления, должны быть инкапсулированы в этот класс и подчинены только этой задаче.
2. OCP (open-closed principle) - программные сущности должны быть открыты для расширения, но закрыты для модификации
3. LSP (Liskov substitution principle)
4. ISP (interface segregation principle) - много интерфейсов, специально предназначенных для клиентов, лучше, чем один интерфейс общего назначения
5. DIP (dependency inversion principle) - зависимость на Абстракциях. Нет зависимости на что-то конкретное

### Касты при private и protected наследовании
```cpp
class Base {};
class Derived : private Base {};

int main() {
	Base& b = d;
	Base bb = d; // CE - про приватное наследование знает только наследник
}
```

### Обратный каст
```cpp
Base b;
Derived& d = static_cast<Derived&>(b); // UB - захватываем лишнюю память
Derived* d_ptr = static_cast<Derived*>(&b); // UB - захватываем лишнюю память
```

# Множественное наследование
`struct C: A, B { ... };`
- Размещение в пямяти работает по следующему принципу: сначала родители в том порядке, в котором они унаследованы, потом наследники. С конструкторами и деструкторами то же самое.
- `d` и `&s` могут численно не совпадать (если классы-родители не пустые) (потому что `Dad` лежит в `Son` правее начала на `sizeof(Mom)`)
### Example
```cpp
#include <iostream>

struct Mom { int a = 1; };
struct Dad { int b = 2; };
struct Son: Mom, Dad { int c = 3; };

int main() {
	Son s;
	std::cout << *(reinterpret_cast<int*>(&s));
	std::cout << *(reinterpret_cast<int*>(&s) + 1);
	std::cout << *(reinterpret_cast<int*>(&s) + 2) << std::endl;
  
	Mom* m = &s;
	std::cout << *(reinterpret_cast<int*>(m));
	std::cout << *(reinterpret_cast<int*>(m) + 1) << std::endl;
  
	Dad* d = &s;
	std::cout << *(reinterpret_cast<int*>(d));
	std::cout << *(reinterpret_cast<int*>(d) + 1) << std::endl;
	// Output: 123, 12, 23
}
```

### Проблема ромбовидного наследования
```cpp
struct Granny { int x; };
struct Mother: Granny { int y; };
struct Dad: Granny { int z; };
struct Son: Mother, Dad { int t;};

int main() {
	Son s;
	s.x;  // CE
}
```


# Dynamic polymorphism
- See [[DynamicPolymorphism]]

# Friend
### Example 1
```cpp
struct S {
  public:
    int x;
  protected:
    int y;
  private:
    int z;
};

struct S1 : private S {
  friend class C;
};

class C {
 public:
  void foo() {
    S s;
    S1 s1;

    s.x;
    s1.x;

    s1.y;
    // s1.z; - private in S is not accessible for S1
    // s.y; - relationship friend-parent is not transitive
  }
};

int main() {
  C c;
  c.foo();
}
```

### Example 2
```cpp
#include <iostream>

struct S {
  public:
    int x;
  protected:
    int y;
  private:
    int z;
};

struct S1 : private S {};

struct S2 : public S1 {
  void foo(::S s) {  // pure S s is CE
    s.x;
    // s.y; CE
  }
};

int main() {
  S2 s;
  s.foo(S{});
}
```

### Example 3
```cpp
#include <iostream>

struct A;
struct B;

struct A {
 protected:
  int x;
};

void foo(const B&, const A&);

struct B : public A {
  friend void foo(const B& b, const A& a) {
    b.x;
    // a.x; CE
  }
};

int main() {

}
```

# Dependent names
Dependent names are characterized by a **dependency** on a template argument.

```cpp
#include <vector>

void NonDependent() {
  //You can access the member size_type directly.
  //This is precisely specified as a vector of ints.

  typedef std::vector<int> IntVector;  
  IntVector::size_type i;

  /* ... */
}

template <class T>
void Dependent() {
  // Now the vector depends on the type T. 
  // Need to use typename to access a dependent name.

  typedef std::vector<T> SomeVector;
  typename SomeVector::size_type i;

  /* ... */
}

int main() {
  NonDependent();
  Dependent<int>();
  return 0;
}
```

This is an example of a peculiar situation regarding the use of dependent names which appears quite frequently. Sometimes the rules governing the use of dependent names are not what one might instinctively expect.
For instance, if you have a dependent class which derives from a depenent base, but within an scope in which a name from the base class apparently doesn't depent on the template, you might get a compiler error just like below.
```cpp
#include <iostream>

template <class T>
class Dependent {
 protected:
  T data;
};

template <class T>
class OtherDependent : public Dependent<T> {
 public:
  void printT() const { 
    std::cout << "T: " << data << std::endl; // ERROR
  }
};

int main() {
  OtherDependent<int> o;
  o.printT();
  return 0;
}
```

This error happens because the compiler will not lookup name `data` inside the base class template since it doesn't dependent on `T` and, consequently, it is not a depedent name. The ways to fix are using `this` or explicitly telling the dependent base class template:

```cpp
std::cout << "T: " << this->data << std::endl; // Ok now.
std::cout << "T: " << Dependent<T>::data << std::endl; // Ok now.
```

or placing `using` declaration:
```cpp
template <class T>
class OtherDependent : public Dependent<T> {
    using Dependent<T>::data; // Ok now.
    ...
};
```

- [Source](https://stackoverflow.com/questions/1527849/how-do-you-understand-dependent-names-in-c)
### Example 1
```cpp
#include <iostream>

template <typename T>
struct Base {
  void foo() {
    std::cout << "Foo\n";
  }
};

template <typename T>
struct Derived : Base<T> {
  void bar() {
    // foo()         // foo - independent name
    Base<T>::foo();  // foo - dependent name
    this->foo();     // this - dependent name
  }
};

int main() {
  Derived<int> d;
  d.bar();
}
```

### Example 2
```cpp
#include <iostream>

void foo() { std::cout << "Global\n"; }

template <typename T>
struct Base {
	void foo() { std::cout << "Base\n"; }
};

// void foo() { std::cout << "Global\n"; } // is the same that foo firstly

template <typename T>
struct Derived : Base<T> {
	void bar() {
		foo(); // foo -independent name
		Base<T>::foo(); // foo - dependent name
		this->foo(); // this - dependent name
	}
};

int main() {
	Derived<int> d;
	d.bar();
}
```
