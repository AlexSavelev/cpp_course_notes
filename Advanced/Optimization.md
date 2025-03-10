# Copy elision
Copy elision is an optimization implemented by most compilers to prevent extra (potentially expensive) copies in certain situations. It makes returning by value or pass-by-value feasible in practice (restrictions apply).
- Simple: Copy elision is a compiler optimization technique that eliminates unnecessary copying/moving of objects.

**C++17**: As of C++17, Copy Elision is guaranteed when an object is returned directly, and in this case, the copy or move constructor need **not** be accessible or present:
```cpp
struct C {
  C() {}
  C(const C&) { std::cout << "A copy was made.\n"; }
};
 
C f() {
  return C();  // Definitely performs copy elision
}

C g() {
    C c;
    return c;  // Maybe performs copy elision
}
 
int main() {
  std::cout << "Hello World!\n";
  C obj = f(); // Copy constructor isn't called
}
```

**Copy elision** is defined in the standard in (`[class.copy]`):
> 31) When certain criteria are met, an implementation is allowed to omit the copy/move construction of a class object, even if the copy/move constructor and/or destructor for the object have side effects. In such cases, the implementation treats the source and target of the omitted copy/move operation as simply two different ways of referring to the same object, and the destruction of that object occurs at the later of the times when the two objects would have been destroyed without the optimization.(*) This elision of copy/move operations, called copy elision, is permitted in the following circumstances (which may be combined to eliminate multiple copies):
> 
> — in a return statement in a function with a class return type, when the expression is the name of a non-volatile automatic object (other than a function or catch-clause parameter) with the same cvunqualified type as the function return type, the copy/move operation can be omitted by constructing the automatic object directly into the function’s return value
> 
> — in a throw-expression, when the operand is the name of a non-volatile automatic object (other than a function or catch-clause parameter) whose scope does not extend beyond the end of the innermost enclosing try-block (if there is one), the copy/move operation from the operand to the exception object (15.1) can be omitted by constructing the automatic object directly into the exception object
> 
> — when a temporary class object that has not been bound to a reference (12.2) would be copied/moved to a class object with the same cv-unqualified type, the copy/move operation can be omitted by constructing the temporary object directly into the target of the omitted copy/move
> 
> — when the exception-declaration of an exception handler (Clause 15) declares an object of the same type (except for cv-qualification) as the exception object (15.1), the copy/move operation can be omitted by treating the exception-declaration as an alias for the exception object if the meaning of the program will be unchanged except for the execution of constructors and destructors for the object declared by the exception-declaration.
> 
> * Because only one object is destroyed instead of two, and one copy/move constructor is not executed, there is still one object destroyed for each one constructed.

The example given is:
```cpp
class Thing {
 public:
  Thing();
  ~Thing();
  Thing(const Thing&);
};

Thing f() {
  Thing t;
  return t;
}

Thing t2 = f();
```

and explained:

> Here the criteria for elision can be combined to eliminate two calls to the copy constructor of class `Thing`: the copying of the local automatic object `t` into the temporary object for the return value of function `f()` and the copying of that temporary object into object `t2`. Effectively, the construction of the local object `t` can be viewed as directly initializing the global object `t2`, and that object’s destruction will occur at program exit. Adding a move constructor to `Thing` has the same effect, but it is the move construction from the temporary object to `t2` that is elided.

- [Source](https://stackoverflow.com/questions/12953127/what-are-copy-elision-and-return-value-optimization/12953145#12953145)
# Common forms of copy elision
- _**Def:**_ RVO - Return Value Optimization
- _**Def:**_ NRVO - Named Return Value Optimization
- [Habr article](https://habr.com/ru/companies/vk/articles/666330/)

### NRVO
- По идее, при вызове функции `f()` создастся локальная переменная `local_var`, затем создасться переменная-результат `result` в виде копии `local_var`. При выходе из функции со стека `local_var` снимается.
- При NRVO же вместо создания `local_var` компилятор сразу создаст  `result` конструктором по умолчанию в точке вызова функции `f()`. А функция `f()` будет выполнять действия сразу с переменной `result`. То есть в этом случае не будет вызван ни конструктор копии, чтобы скопировать `local_var` в `result`, ни деструктор `local_var`.
- Технически это выглядит так:
	1. Компилятор создаёт конструктором по умолчанию до вызова функции `f()` переменную `result`
	2. Затем неявно передаёт в функцию `f()` указатель на `result`
	3. В рамках функции `f()` не создаёт `local_var`, а вместо этого работает с указателем на `result`
	4. в `return` ничего не копируется, поскольку данные уже там

```cpp
#include <iostream>
#include "Verbose.hpp"

Verbose f() {
  std::cout << "1\n";
  Verbose local_var;
  std::cout << "2\n";
  return local_var;
}

int main() {
  std::cout << "0\n";
  auto result = f();
  std::cout << "3\n";
}

```

#### Output with NRVO:
```OUT
0
1
Verbose 1 default constructed
2
3
Verbose 1 destructed
```

### RVO
- RVO - это частный случай NRVO, когда экземпляр возвращаемого класса создаётся прямо в операторе `return`
```cpp
#include <iostream>
#include "Verbose.hpp"

Verbose f() {
  std::cout << "1\n";
  return Verbose();
}

int main() {
  std::cout << "0\n";
  auto result = f();
  std::cout << "2\n";
}

```

#### Output with RVO:
```OUT
0
1
Verbose 1 default constructed
2
Verbose 1 destructed

```
