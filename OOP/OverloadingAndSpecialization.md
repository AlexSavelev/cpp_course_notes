# Overloading of template functions
Основные замечания:
- Частное лучше общего
- Если есть более частная версия, но нужно будет сделать приведение типов, то лучше более общая версия
#### Example 1
```cpp
template <typename T>
void f(T x) { std::cout << 1; }

void f(int x) { std::cout << 2; }  // overloading

int main() {
	f(0); // 2 - более частный случай
}
```

#### Example 2
```cpp
template <typename T, typename U>
void f(T x, U y) { std::cout << 1; }

template <typename T>
void f(T x, T y) { std::cout << 2; }

void f(int x, double y) { std::cout << 3; }

int main() {
	f(0, 0); // 2 (2 лучше 1, т.к. более частная, 2 лучше 3, т.к. не надо делать приведение типов)
}
```

#### Example 3
```cpp
template <typename T>
void f(T x) { std::cout << 1; }

template <typename T>
void f(T& x) { std::cout << 2; }

int main() {
    int x = 0;
    int& y = x;
    f(y); // CE - ambiguous call (см. ссылки)
}
```

#### Example 4
```cpp
template <typename T>
void f(T& x) { std::cout << 1; }

template <typename T>
void f(const T& x) { std::cout << 2; }

int main() {
    int x = 0;
    int& y = x;
    f(y); // 1 - константность еще надо навесить
}
```

# Template specialization
- Syntax: `template <> declaration`
- Any of the following can be fully specialized:
    1. function template
	2. class template
    3. variable template (since C++14)
    4. member function of a class template
    5. static data member of a class template
    6. member class of a class template
    7. member enumeration of a class template
    8. member class template of a class or class template
    9. member function template of a class or class template
    10. member variable template of a class or class template (since C++14) 
- [Source](https://en.cppreference.com/w/cpp/language/template_specialization)

```cpp
template<typename T>  // primary template
struct is_void : std::false_type {};
template<>            // explicit specialization for T = void
struct is_void<void> : std::true_type {};
```

```cpp
template <typename T>
struct Vector {
	void foo() { ... }
};

template <>
struct Vector<bool> {  // Полная специализация шаблонного класса (придется заного написать класс, он не будет иметь ничего общего с Vector<T>)
	std::string g(std::vector<int>&) { ... }
};
```

- Explicit specialization may be declared in any scope where its primary template may be defined (which may be different from the scope where the primary template is defined; such as with out-of-class specialization of a member template). Explicit specialization has to appear after the non-specialized template declaration. 
```cpp
namespace N {
    template<class T> // primary template
    class X { /*...*/ };
    template<>        // specialization in same namespace
    class X<int> { /*...*/ };
 
    template<class T> // primary template
    class Y { /*...*/ };
    template<>        // forward declare specialization for double
    class Y<double>;
}
 
template<> // OK: specialization in same namespace
class N::Y<double> { /*...*/ };
```

- Specialization must be declared before the first use that would cause implicit instantiation, in every translation unit where such use occurs:
```cpp
class String {};
 
template<class T>
class Array { /*...*/ };
 
template<class T> // primary template
void sort(Array<T>& v) { /*...*/ }
 
void f(Array<String>& v) {
    sort(v); // implicitly instantiates sort(Array<String>&), 
}            // using the primary template for sort()
 
template<> // ERROR: explicit specialization of sort(Array<String>)
void sort<String>(Array<String>& v); // after implicit instantiation
```

- A template specialization that was declared but not defined can be used just like any other incomplete type (e.g. pointers and references to it may be used):
```cpp
template<class T> // primary template
class X;
template<>        // specialization (declared, not defined)
class X<int>;
 
X<int>* p; // OK: pointer to incomplete type
X<int> x;  // error: object of incomplete type
```

- Whether an explicit specialization of a function or variable template is inline/constexpr/constinit/consteval is determined by the explicit specialization itself, regardless of whether the primary template is declared with that specifier. Similarly, attributes appearing in the declaration of a template have no effect on an explicit specialization of that template:
```cpp
template<class T>
void f(T) { /* ... */ }

template<>
inline void f<>(int) { /* ... */ } // OK, inline
 
template<class T>
inline T g(T) { /* ... */ }

template<>
int g<>(int) { /* ... */ }         // OK, not inline
 
template<typename>
[[noreturn]] void h([[maybe_unused]] int i);

template<> void h<int>(int i) {
    // [[noreturn]] has no effect, but [[maybe_unused]] has
}
```

- When specializing a function template, its template arguments can be omitted if template argument deduction can provide them from the function arguments:
```cpp
template<class T>
class Array { /*...*/ };
 
template<class T> // primary template
void sort(Array<T>& v);
template<>        // specialization for T = int
void sort(Array<int>&);
 
// no need to write
// template<> void sort<int>(Array<int>&);
```
- An explicit specialization cannot be a friend declaration.

- When defining a member of an explicitly specialized class template outside the body of the class, the syntax template<> is not used, except if it's a member of an explicitly specialized member class template, which is specialized as a class template, because otherwise, the syntax would require such definition to begin with `template<parameters>` required by the nested template:
```cpp
template<typename T>
struct A {
    struct B {};      // member class 
 
    template<class U> // member class template
    struct C {};
};
 
template<> // specialization
struct A<int> {
    void f(int); // member function of a specialization
};

// template<> not used for a member of a specialization
void A<int>::f(int) { /* ... */ }
 
template<> // specialization of a member class
struct A<char>::B {
    void f();
};

// template<> not used for a member of a specialized member class either
void A<char>::B::f() { /* ... */ }
 
template<> // specialization of a member class template
template<class U>
struct A<char>::C {
    void f();
};
 
// template<> is used when defining a member of an explicitly
// specialized member class template specialized as a class template
template<>
template<class U>
void A<char>::C<U>::f() { /* ... */ }
```

- A member or a member template of a class template may be explicitly specialized for a given implicit instantiation of the class template, even if the member or member template is defined in the class template definition.
```cpp
template<typename T>
struct A {
    void f(T);         // member, declared in the primary template

    void h(T) {}       // member, defined in the primary template

    template<class X1> // member template
    void g1(T, X1);

    template<class X2> // member template
    void g2(T, X2);
};

// specialization of a member
template<>
void A<int>::f(int);

// member specialization OK even if defined in-class
template<>
void A<int>::h(int) {}

// out of class member template definition
template<class T>
template<class X1>
void A<T>::g1(T, X1) {}

// member template specialization
template<>
template<class X1>
void A<int>::g1(int, X1);

// member template specialization
template<>
template<>
void A<int>::g2<char>(int, char); // for X2 = char

// same, using template argument deduction (X1 = char)
template<> 
template<>
void A<int>::g1(int, char);
```

- A member or a member template may be nested within many enclosing class templates. In an explicit specialization for such a member, there's a template<> for every enclosing class template that is explicitly specialized.
```cpp
template<class T1>
struct A {
    template<class T2>
    struct B {
        template<class T3>
        void mf();
    };
};
 
template<>
struct A<int>;
 
template<>
template<>
struct A<char>::B<double>;
 
template<>
template<>
template<>
void A<char>::B<char>::mf<double>();
```

- In such a nested declaration, some of the levels may remain unspecialized (except that it can't specialize a class member template in namespace scope if its enclosing class is unspecialized). For each of those levels, the declaration needs `template<arguments>`, because such specializations are themselves templates:
```cpp
template<class T1>
class A {
    template<class T2>
    class B {
        template<class T3> // member template
        void mf1(T3);

        void mf2();        // non-template member
    };
};
 
// specialization
template<>        // for the specialized A
template<class X> // for the unspecialized B
class A<int>::B {
    template<class T>
    void mf1(T);
};
 
// specialization
template<>        // for the specialized A
template<>        // for the specialized B
template<class T> // for the unspecialized mf1
void A<int>::B<double>::mf1(T t) {}
 
// ERROR: B<double> is specialized and is a member template, so its enclosing A
// must be specialized also
template<class Y>
template<>
void A<Y>::B<double>::mf2() {}
```

- Явная инстанциация:
```cpp
#include <iostream>

template <typename T>
T SumOfTwo(const T& lhs, const T& rhs) { return lhs + rhs; }

template int SumOfTwo(const int& lhs, const int& rhs); // явная инстанциация

template <>
int SumOfTwo(const int& lhs, const int& rhs); // объявление специализации - CE (specialization of ‘T SumOfTwo(const T&, const T&) [with T = int]’ after instantiation)

int SumOfTwo(const int& lhs, const int& rhs); // объявление перегрузки

template <typename T>
void bar() {
  SumOfTwo<T>(2, 3);
}

int main() {
  SumOfTwo<int>(3, 2);  
  SumOfTwo<long long>(3, 2);  
  SumOfTwo<double>(3, 2);
  bar<int>();
}
```

# Partial specialization
- Частичная специализация
- Так как у функций существует механизм перегрузки, то понятие частичной специализации к ним не применимо.
- [Source](https://en.cppreference.com/w/cpp/language/partial_specialization)
### Syntax
```cpp
template <typename T>
class Vector<T*> { // Это специализация будет работать для всех T, которые являются указателями
    ...
};
```

```cpp
template <typename T, typename U>
class C {
    ...
};

template <typename T>
class C<T, int> { // работает по тем же правилам, что и перегрузка
    ...
};
```

### Rules
```cpp
template<class T1, class T2, int I>
class A {};             // primary template
 
template<class T, int I>
class A<T, T*, I> {};   // #1: partial specialization where T2 is a pointer to T1
 
template<class T, class T2, int I>
class A<T*, T2, I> {};  // #2: partial specialization where T1 is a pointer
 
template<class T>
class A<int, T*, 5> {}; // #3: partial specialization where
                        //     T1 is int, I is 5, and T2 is a pointer
 
template<class X, class T, int I>
class A<X, T*, I> {};   // #4: partial specialization where T2 is a pointer
```

1) The argument list cannot be identical to the non-specialized argument list (it must specialize something):
```cpp
template<class T1, class T2, int I> class B {};        // primary template
template<class X, class Y, int N> class B<X, Y, N> {}; // error

// Moreover, the specialization has to be more specialized than the primary template
template<int N, typename T1, typename... Ts> struct B;
template<typename... Ts> struct B<0, Ts...> {}; // Error: not more specialized
```

2) Default arguments cannot appear in the argument list
3) If any argument is a pack expansion, it must be the last argument in the list
4) Non-type argument expressions can use template parameters as long as the parameter appears at least once outside a non-deduced context (note that only clang and gcc 12 support this feature currently):
```cpp
template<int I, int J> struct A {};
template<int I> struct A<I + 5, I * 2> {}; // error, I is not deducible
 
template<int I, int J, int K> struct B {};
template<int I> struct B<I, I * 2, 2> {};  // OK: first parameter is deducible
```

5) Non-type template argument cannot specialize a template parameter whose type depends on a parameter of the specialization:
```cpp
template<class T, T t> struct C {}; // primary template
template<class T> struct C<T, 1>;   // error: type of the argument 1 is T,
                                    // which depends on the parameter T
 
template<int X, int (*array_ptr)[X]> class B {}; // primary template

int array[5];
template<int X> class B<X, &array> {}; // error: type of the argument &array is int(*)[X], which depends on the parameter X
```

### Name lookup
Partial template specializations are not found by name lookup. Only if the primary template is found by name lookup, its partial specializations are considered. In particular, a using declaration that makes a primary template visible, makes partial specializations visible as well:
```cpp
namespace N {
    template<class T1, class T2> class Z {}; // primary template
}
using N::Z; // refers to the primary template
 
namespace N {
    template<class T> class Z<T, T*> {};     // partial specialization
}

Z<int, int*> z; // name lookup finds N::Z (the primary template), the
                // partial specialization with T = int is then used
```

### Partial ordering
When a class or variable(since C++14) template is instantiated, and there are partial specializations available, the compiler has to decide if the primary template is going to be used or one of its partial specializations.
1) If only one specialization matches the template arguments, that specialization is used
2) If more than one specialization matches, partial order rules are used to determine which specialization is more specialized. The most specialized specialization is used, if it is unique (if it is not unique, the program cannot be compiled)
3) If no specializations match, the primary template is used
```cpp
// given the template A as defined above
A<int, int, 1> a1;   // no specializations match, uses primary template
A<int, int*, 1> a2;  // uses partial specialization #1 (T = int, I = 1)
A<int, char*, 5> a3; // uses partial specialization #3, (T = char)
A<int, char*, 1> a4; // uses partial specialization #4, (X = int, T = char, I = 1)
A<int*, int*, 2> a5; // error: matches #2 (T = int, T2 = int*, I= 2)
                     //        matches #4 (X = int*, T = int, I = 2)
                     // neither one is more specialized than the other
```
- Informally "A is more specialized than B" means "A accepts a subset of the types that B accepts".

- The function templates are then ranked as if for function template overloading
```cpp
template<int I, int J, class T> struct X {}; // primary template

template<int I, int J>
struct X<I, J, int> {
    static const int s = 1;
}; // partial specialization #1
// fictitious function template for #1 is
// template<int I, int J> void f(X<I, J, int>); #A
 
template<int I>
struct X<I, I, int> {
    static const int s = 2;
}; // partial specialization #2
// fictitious function template for #2 is 
// template<int I>        void f(X<I, I, int>); #B
 
int main() {
    X<2, 2, int> x; // both #1 and #2 match
// partial ordering for function templates:
// #A from #B: void(X<I, J, int>) from void(X<U1, U1, int>): deduction OK
// #B from #A: void(X<I, I, int>) from void(X<U1, U2, int>): deduction fails
// #B is more specialized
// #2 is the specialization that is instantiated
    std::cout << x.s << '\n'; // prints 2
}
```

### Members of partial specializations
The template parameter list and the template argument list of a member of a partial specialization must match the parameter list and the argument list of the partial specialization.
Just like with members of primary templates, they only need to be defined if used in the program.
Members of partial specializations are not related to the members of the primary template.
Explicit (full) specialization of a member of a partial specialization is declared the same way as an explicit specialization of the primary template.
```cpp
template<class T, int I> // primary template
struct A {
    void f(); // member declaration
};
 
template<class T, int I>
void A<T, I>::f() {}     // primary template member definition
 
// partial specialization
template<class T>
struct A<T, 2> {
    void f();
    void g();
    void h();
};
 
// member of partial specialization
template<class T>
void A<T, 2>::g() {}

// explicit (full) specialization
// of a member of partial specialization
template<>
void A<char, 2>::h() {}
 
int main() {
    A<char, 0> a0;
    A<char, 2> a2;
    a0.f(); // OK, uses primary template’s member definition
    a2.g(); // OK, uses partial specialization's member definition
    a2.h(); // OK, uses fully-specialized definition of
            // the member of a partial specialization
    a2.f(); // error: no definition of f() in the partial
            // specialization A<T,2> (the primary template is not used)
}
```

If a primary template is a member of another class template, its partial specializations are members of the enclosing class template. If the enclosing template is instantiated, the declaration of each member partial specialization is instantiated as well (the same way declarations, but not definitions, of all other members of a template are instantiated).
If the primary member template is explicitly (fully) specialized for a given (implicit) specialization of the enclosing class template, the partial specializations of the member template are ignored for this specialization of the enclosing class template.
If a partial specialization of the member template is explicitly specialized for a given (implicit) specialization of the enclosing class template, the primary member template and its other partial specializations are still considered for this specialization of the enclosing class template.
```cpp
template<class T> struct A {  // enclosing class template
    template<class T2>
    struct B {};      // primary member template

    template<class T2>
    struct B<T2*> {}; // partial specialization of member template
};

template<>
template<class T2>
struct A<short>::B {}; // full specialization of primary member template
                       // (will ignore the partial)

A<char>::B<int*> abcip;  // uses partial specialization T2=int
A<short>::B<int*> absip; // uses full specialization of the primary (ignores partial)
A<char>::B<int> abci;    // uses primary
```

### Example 1
```cpp
template <typename T, typename U>
struct S {};

template <typename T>
struct S <int, T> {};

template <typename T, typename... Args>
struct S <T, std::tuple<Args...>> {};
```

### Example 2
```cpp
#include <iostream>

template <typename T, typename U>
struct FooImpl {
  static void call(T, U) {
    std::cout << 1 << '\n';
  }
};

template <typename T, typename U>
struct FooImpl<T, const U> {
  static void call(T, const U) {
    std::cout << 2 << '\n';
  }
};

template <typename T, typename U>
void foo(T& t, U& u) {
  return FooImpl<T, U>::call(t, u);
}

int main() {
  int x;
  const int y = 0;
  foo(x, x);  // 1
  foo(x, y);  // 2
}

```
# Examples

### Example 1
- _**Note:**_ Вторая функция является уже перегрузочной, поскольку с `f<T, U>` ничего общего не имеет
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

template <typename T>
void f(T, T) { std::cout << 2; }

template <>
void f(int, int) { std::cout << 3; }

int main() {
	f(0, 0); // 3 (среди перегрузок мы выбрали вторую, так как она более частная. А у нее уже выбрали специализацию)
}
```

### Example 2
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

template <>
void f(int, int) { std::cout << 3; }

template <typename T>
void f(T, T) { std::cout << 2; }

int main() {
	f(0, 0); // 2 (Специализация применилась к 1 функции. На моменте выбора перегрузки выиграла 2, так как она более частная. У 2 версии нет специализации, поэтому она вызвалась сама
}
```

### Example 3
```cpp
template <typename T, typename U>
void f(T, U) { std::cout << 1; }

// тут template убрали
void f(int, int) { std::cout << 3; }

template <typename T>
void f(T, T) { std::cout << 2; }

int main() {
	f(0, 0); // 3 (потому что теперь это тоже перегрузка)
}
```

### Example 4
```cpp
#include <iostream>

template <typename T>
void foo(T) {
  std::cout << 1 << '\n';
}

template <typename T>
void foo(T*) {
  std::cout << 2 << '\n';
}

template <>
void foo(int) {
  std::cout << 3 << '\n';
}

int main() {
  foo(10);  // 3
}
```

### Example 5
```cpp
#include <iostream>

template <typename T, typename U>
void bar(T, U) {
  std::cout << 1 << '\n';
}

template <typename T, typename U>
void bar(T*, U*) {
  std::cout << 2 << '\n';
}


template <>
void bar<int*, int*>(int*, int*) {
  std::cout << 3 << '\n';
}  // Эта функция - специализация первой функции (не второй), Если убрать <int*, int*>, но выберет под основу то вторую. Тест на внимательность

int main() {
  int* ptr = nullptr;
  bar(ptr, ptr);  // 2
}
```

### Example 6
```cpp
#include <iostream>

template <typename T, typename U>
void baz(T, U) {
  std::cout << 1 << '\n';
}

template <>
void baz(int*, int*) {
  std::cout << 2 << '\n';
}

template <typename T, typename U>
void baz(T*, U*) {
  std::cout << 3 << '\n';
}

template <>
void baz(int*, int*) {
  std::cout << 4 << '\n';
}

int main() {
  int* ptr = nullptr;
  baz(ptr, ptr);  // 4 - выберет специализацию (4) наиболее подходящего шаблона (3)
  baz<int*, int*>(ptr, ptr);  // 2
}
```

### Example 7
```cpp
#include <iostream>

template <typename T>
void foo(T) {
  std::cout << "T\n";
}

struct S {};

template <typename T>
void call(T t, S s) {
  foo(t);
  foo(s);
}

void foo(S) {
  std::cout << "S\n";
}

int main() {
  S s;
  call(s, s);  // ST
  // s - независимо от шаблона, для foo(s) существует только шаблон
  // t - зависимо от шаблона, зарезолвить foo(t) получится уже после того, как найдется более подходящая перегрузка
}
```

### Example 8
```cpp
#include <iostream>

template <typename T>
void foo() {}

template <>
void foo<int>() = delete;

int main() {
  foo<void>();
  // foo<int>();  // CE
}
```

