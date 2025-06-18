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

- Заводится специальная таблица в статической памяти, в которой для каждого типа написано, на какой сдвиг от начала объекта нужно переместиться чтобы найти бабушку
- Этот указатель указывает на запись в этой таблице
- _**Def**_ Эта таблица называется vtable

# Basics of dynamic polymorphism
- _**Def**_ Полиморфизм - это способность функции обрабатывать данные разных типов
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

- Рассмотрим еще пример:
```cpp
struct Base {
  virtual void f() { std::cout << 1; }
};

struct Derived: Base {
  void f() const {std::cout << 2; }
};

int main() {
  Derived d;
  Base& b = d;
  b.f();
}
```
- `f` в `Derived` не является виртуальной, потому что не совпадает сигнатура
- То есть механизм виртуальных функций работает только при полном совпадении сигнатур
- Если навесить на вторую `f` `override`, будет CE

- Поэтому, чтобы не ошибаться, можно (и даже нужно) использовать ключевое слово `override`

### `final` and `override`
- Существует ключевое слово `final`, которое запрещает наследникам переопределять эту виртуальную функцию

- [Source](https://stackoverflow.com/questions/29412412/does-final-imply-override)
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

#### Expanded example
```cpp
#include <iostream>
using std::cout; using std::endl;

struct B {
  virtual void f1() { cout << "B::f1() "; }
  virtual void f2() { cout << "B::f2() "; }
  virtual void f3() { cout << "B::f3() "; }
  virtual void f6() final { cout << "B::f6() "; }
  void f7() { cout << "B::f7() "; }
  void f8() { cout << "B::f8() "; }
  void f9() { cout << "B::f9() "; }
};

struct D : B {
  void f1() override { cout << "D::f1() "; }
  void f2() final { cout << "D::f2() "; }
  void f3() override final { cout << "D::f3() "; }  // need not have override
  // should have override, otherwise add new virtual function
  virtual void f4() final { cout << "D::f4() "; }
  //virtual void f5() override final;  // Error, no virtual function in base class
  //void f6(); // Error, override a final virtual function
  void f7() { cout << "D::f7() "; }
  virtual void f8() { cout << "D::f8() "; }
  //void f9() override;  // Error, override a nonvirtual function 
};

int main() {
  B b; D d;
  B *bp = &b, *bd = &d; D *dp = &d;
  bp->f1(); bp->f2(); bp->f3(); bp->f6(); bp->f7(); bp->f8(); bp->f9(); cout << endl;
  bd->f1(); bd->f2(); bd->f3(); bd->f6(); bd->f7(); bd->f8(); bd->f9(); cout << endl;
  dp->f1(); dp->f2(); dp->f3(); dp->f6(); dp->f7(); dp->f8(); dp->f9(); cout << endl;
  return 0;
}
```

```OUTPUT
B::f1() B::f2() B::f3() B::f6() B::f7() B::f8() B::f9()
D::f1() D::f2() D::f3() B::f6() B::f7() B::f8() B::f9()
D::f1() D::f2() D::f3() B::f6() D::f7() D::f8() B::f9()
```

1. Compare `f1()` and `f6()`. We know that `override` and `final` is indepent sematically.
    - `override` means the function is overriding a virtual function in its base class. See `f1()` and `f3()`.
    - `final` means the function cannot be overrided by its derived class. (But the function itself need not override a base class virtual function.) See `f6()` and `f4()`.
2. Compare `f2()` and `f3()`. We know that if a member function is declared without `virtual` and with `final`, it means that it already override a virtual function in base class. In this case, the key word `override` is redundant.
3. Compare `f4()` and `f5()`. We know that if a member function is declared with `virtual`and if it is not the _first_ virtual function in inheritance hierarchy, then we should use `override` to specify the override relationship. Otherwise, we may accidentally add new virtual function in derived class.
4. Compare `f1()` and `f7()`. We know that any member function, not just virtual ones, can be overridden in derived class. What `virtual` specifies is _polymorphism_, which means the decision as to which function to run is delayed until run time instead of compile time. (This should be avoid in practice.)
5. Compare `f7()` and `f8()`. We know that we can even override a base class function and make it a new virtual one. (Which means any member function `f8()` of class derived from `D` will be virtual.) (This should be avoid in practice too.)
6. Compare `f7()` and `f9()`. We know that `override` can help us find the error when we want to override a virtual function in derived class while forgot to add key word `virtual` in base class.

#### Conclusion
In conclusion, the best practice is:
- _only_ use `virtual` in declaration of the _first_ virtual function in base class;
- always use `override` to specify override virtual function in derived class, unless `final` is also specified.

#### Final classes and structs
- `final` еще может быть использовано в наследовании, чтобы запретить дальше наследоваться: `struct B final : A { ... };`
```cpp
// ...
struct Derived final : Base {
  // ...
};
```

#### Final profit
- С методами/классами, подмеченными как `final` работа окажется в разы быстрее: компилятор видит "финальность" и не находит необходимым лезть в vtable.

### Abstract classes
- Это классы, в которых есть хотя бы один pure virtual метод (`virtual void foo() = 0;`)

Для абстрактного класса верно следующее:
1. Создать объект абстрактного класса нельзя (CE)
2. Можно заводить указатели и ссылки на абстрактный класс
3. Если наследник не переопределит все чисто виртуальные методы родителя, то наследник тоже будет считаться абстрактным классом
#### Example
- При этом можно написать реализацию pure virtual метода, однако класс в этом случае все равно не перестает быть абстрактным

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

- Если же убрать определение `Animal::Say`, то будет ошибка
```OUT
/usr/bin/ld: /tmp/ccvvm60A.o: in function `main':
main.cpp:(.text+0x25f): undefined reference to `Animal::Say()'
collect2: error: ld returned 1 exit status
```

### Virtual destructor
- Must have
	- Если в наследнике выделяется динамическая память, которая чиститься как раз в деструкторе, то, работая с этим классом от имени родителя, можем допустить утечку памяти

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
```cpp
#include <iostream>

struct S {
  virtual void f(int a = 10) = 0;
};

struct C : S {
  void f(int a = 20) override { std::cout << a << '\n'; }
};

int main() {
  C c;
  c.f();  // 20

  S& s = c;
  s.f();  // 10
}
```

# RTTI
- Run Time Type Identification
	- Динамическая идентификация типов данных
- Проблема: невозможно в compile time определить дочерний тип

_**Def**_ Тип называется полиморфным, если у него есть хотя бы одна виртуальная функция

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
VTable record: `[ptr to type_info][&Base::f]`
- `type_info` при этом хранится где-то в статической памяти

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

- При этом:
```cpp
Son s;
Granny& g = s;
dynamic_cast<Son&>(g); // CE, source type is not polymorphic
```

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
