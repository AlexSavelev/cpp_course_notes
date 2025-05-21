# Примитивы метапрограммирования
```cpp
template <typename T, T x>
struct integral_constant {
    static constexpr T value = x;
};

template <bool x>
using bool_constant = integral_constant<bool, b>;

using true_type = bool_constant<true>;
using false_type = bool_constant<false>;
```

```cpp
template <typename T, typename U>
struct is_same: false_type {};

template <typename T>
struct is_same<T, T>: true_type {};  // specialization

template <typename T, typename U>
using is_same_v = is_same<T, U>::value;  // alias
```

```cpp
template <typename... Types>
struct conjuction: true_type {};  // base

template <typename Type>
struct conjuction<Type>: Type {};  // one argument

template <typename Type, typename... Types>
struct conjuction<Type, Types...>: std::conditional_t<Type::value, conjuction<Types...>, Type> {};  // if value is true => calculate others, is false => false
```

# SFINAE
- Substitution failure is not an error
- Неудачная шаблонная подстановка не является ошибкой компиляции
- Непосредственный контекст подстановки
	- ==TODO==

```cpp
#include <iostream>

template <typename T>
auto f(const T&) {
  std::cout << 1;
  return 0;
}

auto f(...) {  // style from C (accept any argument(s))
  std::cout << 2;
  return 0;
}

int main() {
  f(5);  // 1
}
```

```cpp
#include <iostream>

template <typename T>
auto f(const T&) -> decltype(T().size()) {
  std::cout << 1;
  return 0;
}

auto f(...) {
  std::cout << 2;
  return 0;
}

int main() {
  f(5);  // 2
}
```

- Собственно, тут и происходит SFINAE. При подставновке `int` в первую версию функции возникает ошибка подстановки (нельзя сделать `int().size()`), но это не ошибка компиляции и компилятор пытается выбрать следующую перегрузку
- _**Note**_ Это работает только если ошибка возникает именно во время подстановки в сигнатуру функции, а не в тело

- Можно сломать либо `return_type`, либо `argument list`, либо `template parameter list`
```cpp
template <typename T, typename = decltype(T().size())>  // отбросили имя второго параметра ввиду его ненадобности
size_t f(const T&) {
  std::cout << 1;
  return 0;
}

auto f(...) {
  std::cout << 2;
  return 0;
}
```

# `std::enable_if`
- Очень часто мы хотим выключать какие-то перегрузки в зависимости от типа `T`. В этом нам поможет `enable_if`
- Пример: хотим включать перегрузку только если `T` это класс. Воспользуемся метаклассом `std::is_class`
```cpp
#include <iostream>
#include <type_traits>

template <typename T, typename = std::enable_if_t<std::is_class_v<std::remove_reference_t<T>>>>
void g(const T&) {
  std::cout << 1;
}

template <typename T, typename = std::enable_if_t<!std::is_class_v<std::remove_reference_t<T>>>, typename = void>  // последнее ввели, чтобы исключить ошибку о перегрузке с одинаковой сигнатурой
void g(const T&) {
  std::cout << 2;
}
```

```cpp
#include <iostream>
#include <type_traits>

template <typename T, std::enable_if_t<std::is_class_v<std::remove_reference_t<T>>, bool> = true>
void g(const T&) {
  std::cout << 1;
}

template <typename T, std::enable_if_t<!std::is_class_v<std::remove_reference_t<T>>, bool> = true>
void g(const T&) {
  std::cout << 2;
}
```
==TODO== `T&&` приоритетнее чем `const T&`

### Possible implementation
- Работает достаточно просто: при использовании `enable_if` мы обращаемся к `enable_if::type`. В случае `false` внутри `enable_if` `type` нет => ошибка шаблонной подстановки => SFINAE
- Во втором примере у нас есть две шаблонные функции с аргументами `typename T`, и неким `std::enable_if_t` (который превращается либо во что-то невалидное, либо в `bool`)
- Но с точки зрения шаблонной сигнатуры последний аргумент разный
```cpp
template <bool B, typename = void>
struct enable_if {};

template <typename T>
struct enable_if<true, T> {
  using type = T;
};

template <bool B, typename T = void>
using enable_if_t = typename enable_if<B,T>::type;
```

# `declval`
- Для типа `T` мы хотим иметь выражение которое будет иметь тип `T`, при этом у типа `T` может отсутствовать конструктор по умолчанию
- Для этого существует функция `declval` (`#include <utility>`)
```cpp
template <typename T>
T declval() noexcept;
```

- Вообще реализация выглядит следующим образом
```cpp
template<typename T>
typename std::add_rvalue_reference<T>::type declval() noexcept {
    static_assert(false, "declval not allowed in an evaluated context");
}
```
- Возврат rvalue reference позволяет определять `declval` в т.ч. и для incomplete-типов (абстрактные классы, просто `struct S;`), и тех, где не определен конструктор копирования и тд.
	- `T&&` нельзя, потому что при `T = [void]` будет invalid

### Практика. `AddLvalueReference`
```cpp
#include <iostream>
#include <type_traits>

template <typename T>
T&& declval() noexcept {
  static_assert(0, "Cringe");
};

template <typename... Ts>
using VoidT = void;

template <typename T, typename U = void>
struct AddLvalueReferenceHelper {
  using type = T;
};

template <typename T>
struct AddLvalueReferenceHelper<T, VoidT<T&>> {
  using type = T&;
};

template <typename T>
struct AddLvalueReference : AddLvalueReferenceHelper<T> {};

template <typename T>
using AddLvalueReferenceT = typename AddLvalueReference<T>::type;

int main() {
  std::cout << std::is_same_v<AddLvalueReferenceT<void>, void>;
  std::cout << std::is_same_v<AddLvalueReferenceT<int>, int&>;
}
```

# `has_method_construct`
- Хотим проверить, есть ли у произвольного класса метод `construct` с данными аргументами

```cpp
template <typename T, typename... Args>
struct has_method_construct {
 private:
  template <typename TT, typename... AArgs>
  static auto f(int) -> decltype(declval<TT>().construct(declval<AArgs>()...), int());  // operator, (evaluates L and R then returns R)

  template <typename...>
  static char f(...);

 public:
  static const bool value = std::is_same_v<decltype(f<T, Args...>(0)), int>;
};

template <typename T, typename... Args>
const bool has_method_construct_v = has_method_construct<T, Args...>::value;
```

### Практика: проверяем наличие оператора сравнения
```cpp
#include <iostream>

struct EqCmp {
  friend bool operator==(const EqCmp&, const EqCmp&) = default;
};

struct EqNotCmp {
  friend bool operator==(const EqNotCmp&, const EqNotCmp&) = delete;
};

template <typename T>
struct IsEqualityComparable {
  using yes = char[1];
  using no = char[2];

  template <typename U = T>
  static decltype(std::declval<const U&>() == std::declval<const U&>(), yes{})&
  helper(int) {}

  static no& helper(...) {}

  enum { value = sizeof(helper(4)) == sizeof(yes) };
};

template <typename T>
bool is_equality_comparable_v = IsEqualityComparable<T>::value;

int main() {
  std::cout << is_equality_comparable_v<int>;
  std::cout << is_equality_comparable_v<std::string>;
  std::cout << is_equality_comparable_v<EqCmp>;
  std::cout << is_equality_comparable_v<EqNotCmp>;

  return 0;
}
```

# `move_if_noexcept`
- Вспоминаем, что было в векторе
```cpp
template <typename T>
void Vector<T>::reserve(size_t n) {
  if (n <= capacity_) { return; }
  T* new_arr = AllocTraits::allocate(alloc_, n);
  size_t i = 0;
  try {
    for (;i < size_; ++i) {
      AllocTraits::construct(alloc_, new_arr + i, std::move(arr_[i])); // тут нужно копировать если std::move кидает ошибку
    }
  } catch (...) {
    for (size_t j = 0; j < i; ++j) {
      AllocTraits::destroy(alloc_, new_arr + j);
    }
    AllocTraits::dallocate(alloc_, n);
    throw;
  }
  for (size_t i = 0; i < size_; ++i) { AllocTraits::destroy(alloc_, arr_ + i); }
  AllocTraits:deallocate(alloc_, arr_, capacity_)
  arr_ = new_arr;
  capacity_ = n;
}
```
- При перемещении с `std::move` у нас нет сильной гарантии безопасности, поэтому будем использовать метод `move_if_noexcept`
- Вся задумка в возвращаемом значении: если все хорошо и конструктор перемещения `noexcept`, то мы вернем `T&&`, а если нет, то `const T&`
- А вообще `noexcept(false)` конструктор перемещения и оператор перемещения - это кринж, который приравнивают с деструктором, который может выбрасывать исключения

### Implementation
```cpp
template<typename T>
struct move_if_noexcept_cond
: public std::conjuction<std::negation<is_nothrow_move_constructible<T>>,
                         is_copy_constructible<T>>>
{ };

template<typename T>
constexpr typename conditional<move_if_noexcept_cond<std::remove_reference_t<T>>::value,
            const std::remove_reference_t<T>&,
            std::remove_reference_t<T>&&
           >
move_if_noexcept(T& x) noexcept
{ return std::move(x); }
```

# `std::is_base_of`
### Implementation 1
```cpp
template <typename B, typename D>
std::true_type test_is_base_of(const B*);

template <typename Args...>
std::false_type test_is_base_of(...);

template <typename B, typename D>
struct is_base_of: std::conjuction<
                    std::is_class<B>,
                    std::is_class<D>,
                    decltype(test_is_base_of<B, D>(static_cast<D*>(nullptr)))
                   >;
```

### Implementation 2
```cpp
namespace details {
    template<typename B>
    std::true_type test_ptr_conv(const volatile B*);

    template<typename>
    std::false_type test_ptr_conv(const volatile void*);
 
    template<typename B, typename D>
    auto test_is_base_of(int) -> decltype(test_ptr_conv<B>(static_cast<D*>(nullptr)));

    template<typename, typename>
    auto test_is_base_of(...) -> std::true_type; // private or ambiguous base (похоже уровень приватности проверяется в очень удобный для этого момент)
}
 
template<typename Base, typename Derived>
struct is_base_of : std::integral_constant<
        bool,
        std::is_class<Base>::value &&
        std::is_class<Derived>::value &&
        decltype(details::test_is_base_of<Base, Derived>(0))::value
    > {};
```

```cpp
#include <type_traits>
 
class A {};
class B : A {};
class C : B {};
class D {};
union E {};
using I = int;
 
static_assert
(
    std::is_base_of_v<A, A> == true &&
    std::is_base_of_v<A, B> == true &&
    std::is_base_of_v<A, C> == true &&
    std::is_base_of_v<A, D> != true &&
    std::is_base_of_v<B, A> != true &&
    std::is_base_of_v<E, E> != true &&
    std::is_base_of_v<I, I> != true
);

int main() {}
```

# Type switch
- [Source](https://stackoverflow.com/questions/22822836/type-switch-construct-in-c11)

### Implementation
```cpp
#include <cassert>
#include <functional>
#include <type_traits>

template<typename T> struct remove_class { };
template<typename C, typename R, typename... A>
struct remove_class<R(C::*)(A...)> { using type = R(A...); };
template<typename C, typename R, typename... A>
struct remove_class<R(C::*)(A...) const> { using type = R(A...); };
template<typename C, typename R, typename... A>
struct remove_class<R(C::*)(A...) volatile> { using type = R(A...); };
template<typename C, typename R, typename... A>
struct remove_class<R(C::*)(A...) const volatile> { using type = R(A...); };

template<typename T>
struct get_signature_impl { using type = typename remove_class<
    decltype(&std::remove_reference<T>::type::operator())>::type; };
template<typename R, typename... A>
struct get_signature_impl<R(A...)> { using type = R(A...); };
template<typename R, typename... A>
struct get_signature_impl<R(&)(A...)> { using type = R(A...); };
template<typename R, typename... A>
struct get_signature_impl<R(*)(A...)> { using type = R(A...); };
template<typename T> using get_signature = typename get_signature_impl<T>::type;


template<typename Base, typename FirstSubclass, typename... RestOfSubclasses>
void typecase(
        Base *base,
        FirstSubclass &&first,
        RestOfSubclasses &&... rest) {

    using Signature = get_signature<FirstSubclass>;
    using Function = std::function<Signature>;

    if (typecaseHelper(base, (Function)first)) {
        return;
    }
    else {
        typecase(base, rest...);
    }
}
template<typename Base>
void typecase(Base *) {
    assert(false);
}
template<typename Base, typename T>
bool typecaseHelper(Base *base, std::function<void(T *)> func) {
    if (T *first = dynamic_cast<T *>(base)) {
        func(first);
        return true;
    }
    else {
        return false;
    }
}
```

### Usages
```cpp
#include <iostream>

class MyBaseClass {
public:
    virtual ~MyBaseClass() { }
};
class MyDerivedA : public MyBaseClass { };
class MyDerivedB : public MyBaseClass { };


int main() {
    MyBaseClass *basePtr = new MyDerivedB();

    typecase(basePtr,
        [](MyDerivedA *a) {
            std::cout << "is type A!" << std::endl;
        },
        [](MyDerivedB *b) {
            std::cout << "is type B!" << std::endl;
        });

    return 0;
}
```
