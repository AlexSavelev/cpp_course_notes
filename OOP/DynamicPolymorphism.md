# Virtual inheritance

```cpp
struct Granny { int g; };
struct Mom: public virtual Granny { int m; };
struct Dad: public virtual Granny { int d; };
struct Son: Mom, Dad { int s;};

// sizeof(Son) == 40;
```
### `Son` object structure

| ptr | mom | ptr | dad | son | granny |
| --- | --- | --- | --- | --- | ------ |
| 8   | 4   | 8   | 4   | 4   | 4      |

- Также заводится специальная таблица в статической памяти, в которой для каждого типа написано, на какой сдвиг от начала объекта нужно переместиться чтобы найти бабушку
- Этот указатель указывает на запись в этой таблице
- _**Def**_ Эта таблица называется vtable

# Basics 
- Статический полиморфизм - перегрузка функций, шаблоны
- Сейчас же вводится динамический
### Solution
```cpp
struct Animal {
	virtual void Say() {
		std::cout << "Abstract animal don't speak\n";
	}
};

struct Dog: Animal {
	void Say() override {
		std::cout << "Woof woof\n";
	}
};

struct Cat: Animal {
	void Say() override {
		std::cout << "Meow\n";
	}
};

void MakeAnimalTalk(Animal& animal) {
	animal.Say();
}

int main() {
	Dog d; MakeAnimalTalk(d); // Woof woof
	Cat c; MakeAnimalTalk(c); // Meow
}
```

- Ключевое слово `virtual` необходимо и достаточно писать только в родителе
- Если написать `virtual` еще и в наследнике то ничего не поменяется
- Если написать `virtual` только в наследнике, то работать будет только для его наследников
- Работает не только с ссылками но и с указателями
- Механизм виртуальных функций работает только при полном совпадении сигнатур
- Существует ключевое слово `final`, которое запрещает наследникам переопределять эту виртуальную функцию

### `final` and `override`
- [Source](https://stackoverflow.com/questions/29412412/does-final-imply-override)[S](https://github.com/isocpp/CppCoreGuidelines/issues/423)
- See the examples:
```cpp
        void f()          final;  // (1)
        void f() override final;  // (2)
virtual void f() override final;  // (3)

virtual void f()          final;  // (4)
```
- For _**(1)**_, `final` always requires a virtual function, and for `f` to be implicitly virtual, it must be overriding a virtual base class function. Hence _**(1)**_ and _**(3)**_ are equivalent.
- _**(2)**_ obviously implies `virtual` so it is also equivalent to _**(1)**_/_**(3)**_.
- _**(4)**_, however, is _not_ equivalent to any of the above. It simply does not require a virtual base class version of `f` for the reasons that _**(1)**_ does. (Which would also be pointless, if it did not have one.)


### Abstract classes
- Это классы, в которых есть хотя бы один `pure virtual` метод (`virtual void foo() = 0;`)

Для абстрактного класса верно следующее:
1. Создать объект абстрактного класса нельзя (CE)
2. Можно заводить указатели и ссылки на абстрактный класс
3. Если наследник не переопределит все чисто виртуальные методы родителя, то наследник тоже будет считаться абстрактным классом
#### Example
```cpp
#include <iostream>

struct Animal {
	virtual void Say() = 0;
};

void Animal::Say() { std::cout << "...\n"; }

struct Dog : Animal {
	void Say() override { std::cout << "Gav!\n"; }
};

int main() {
	// Animal a;  // CE
	Dog dog;
	dog.Say();  // Gav!
	dog.Animal::Say();  // ...
}
```

Класс все равно не перестает быть абстрактным.

### Virtual destructor
- Must have

```cpp
struct Base {
  virtual ~Base() = default;
};

struct Derived {
  Derived() {
    array = new int[10];
  }

  ~Derived() {
    delete[] array;
  }

  int* array;
};

int main() {
  Base* b = new Derived();
  delete b;
}
```

### Аргументы по умолчанию
- Аргумент по умолчанию берется и того класса, из которого мы метод вызываем

# RTTI
- Run Time Type Identification
- Невозможно в compile time определить дочерний тип

_**Def**_ Тип называется полиморфным если у него есть хотя бы одна виртуальная функция

### `dynamic_cast`
- Умеет работать только с полиморфными типами
- Умеет кастить ссылки и указатели. При ссылках при неудачном касте будет вызываться исключение, при указателях - возвращается `nullptr`

```cpp
Derived* ptr = dynamic_cast<Derived*>(&base);
```

- `dynamic_cast` умеет кастовать "вбок"
	- При этом полиморфным должен быть тип, от которого мы кастим (в данном случае `Mother`):
```cpp
struct Mother {
    virtual ~Mother() = default;
};

struct Father {};

struct Son: Mother, Father {
    virtual ~Son() = default;
};

int main() {
  Son s;
  Mother& m = s;
  Father& d = dynamic_cast<Father&>(m);
}
```

# Оператор `typeid`
- Работает для всех типов
- Для полиморфных выводит "настоящий тип"
- Возвращает [std::type_info](https://en.cppreference.com/w/cpp/types/type_info)
- Можно выводить название типа:  `typeid(obj).name()`

# Устройство virtual functions

### Base

```cpp
struct Base {
    virtual void f() {}
    void h();
    int x = 0;
};
```

`sizeof(Base)` = 16
Base structure: `[ptr to vtable][x][4 bytes of padding]`
- ==TODO== padding
VTable record: `[ptr to type_info][&Base::f]`

### Derived

```cpp
struct Derived: Base {
    void f() override {}
    virtual void g();
    int y;
};
```

Derived structure: `[ptr to vtable][x][y]`
VTable record: `[ptr to type_info][&Derived::f][&Derived::g]`

### Multi-inheritance

```cpp
struct Granny {
    int g;
};

struct Mom: Granny {
    virtual void f() {}
    int m;
};

struct Son: Mom {
    void f() override {}
    int s;
};
```

Son structure: `[ptr to vtable][g][m][s]`

# Default arguments for virtual functions
- The function declaration (and its defaults) that is considered is the one of the object class known by the compiler. So the same function of the same object may have different default arguments, depending on the object type used to invoke it
- [Source](https://stackoverflow.com/questions/53272510/default-value-parameter-in-pure-virtual-function)
- Order:
	- Статическое связывание (аргументы по умолчанию)
	- Динамическое связывание (виртуальные методы)

```cpp
#include <iostream>

struct A {
	virtual void f(int a = 1) = 0;
};

struct B : A {
	void f(int a) override;
};

struct C : A {
	void f(int a = 2) override;
};

void B::f(int a) { // definition
	std::cout << "B" << a << std::endl;
}

void C::f(int a) { // definition
	std::cout << "C" << a << std::endl;
}

int main() {
	B b;
	C c;
	A *x = &c,
	  *y = &b; // points to the same objects but using a polymorphic pointer
	x->f(); // default value defined for A::f() but with the implementation of C
	y->f(); // default value defined for A::f() but with the implementation of B
	// b.f(); // default not defined for B::f() so cannot compile
	c.f(); // default value defined for C::f();

	// Output: C1 B1 C2
}
```

### Example 1
```cpp
#include <iostream>

struct Granny {
  Granny(int x = 10) {
    std::cout << x << '\n';
  } 
};

struct Mom : virtual Granny {
  Mom() : Granny(2) {}
};

struct Dad : virtual Granny {
  Dad() : Granny(5) {}
};

struct Son : Mom, Dad {};

struct A { A() { std::cout << "A\n"; } };
struct B { B() { std::cout << "B\n"; } };
struct C { C() { std::cout << "C\n"; } };

struct D : A, virtual B, C {};

int main() {
  Son s;  // 10
  D d;  // B A C
}
```

# NVI
- Non-Virtual Interface
- The essence of the non-virtual interface pattern is that you have **private virtual** functions, which are called by **public non-virtual** functions (the non-virtual interface).
- The advantage of this is that the base class has more control over its behaviour than it would if derived classes were able to override any part of its interface. In other words, the base class (the interface) can provide more guarantees about the functionality it provides.
- [Source](https://stackoverflow.com/questions/6481260/non-virtual-interface-design-pattern-in-c-c)

### Bad solution
```cpp
class Animal
{
public:
    virtual void speak() const = 0;
};

class Dog : public Animal
{
public:
    void speak() const { std::cout << "Woof!" << std::endl; }
};

class Cat : public Animal
{
public:
    void speak() const { std::cout << "Meow!" << std::endl; }
};
```
This uses the usual public virtual interface that we're used to, but it has a couple of problems:
1. Each derived animal is repeating code -- the only part that changes is the string, yet each derived class needs the whole `std::cout << ... << std::endl;` boilerplate code.
2. The base class can't make guarantees about what `speak()` does. A derived class may forget the new line, or write it to `cerr` or anything for that matter.
### Good solution
```cpp
class Animal {
public:
   void speak() const { std::cout << getSound() << std::endl; }
private:
   virtual std::string getSound() const = 0;
};

class Dog : public Animal {
private:
   std::string getSound() const { return "Woof!"; }
};

class Cat : public Animal {
private:
   std::string getSound() const { return "Meow!"; }
};
```
### Example
```cpp
#include <iostream>

// NVI - Non Virtual Interface

struct A {
 public:
  void foo(int x = 100) {
    fooImpl(x);   
  }

 private:
  virtual void fooImpl(int x) {
    std::cout << x << '\n';
  }
};

struct B : A {
 private:
  void fooImpl(int x) override {
    std::cout << x + 1 << '\n';
  }
};

int main() {
  A a;
  B b;
  A& ref = b;
  ref.foo();
}
```



==TODO==
CHECK [FUCK](https://stackoverflow.com/questions/860339/what-is-the-difference-between-public-private-and-protected-inheritance)
