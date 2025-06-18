- Появились в C++20
# Problem overview
- Рассмотрим код
```cpp
template <typename R, typename T>
bool contains(const R& range, const T& value) {
  for (const auto& x : range) {
    if (x == value) {
      return true;
    }
  }
  return false;
}
```
- Понятно, что для `T` и `R` накладываются какие-либо условия (`R::begin/end` с возможностью инкремента и разыменования, `R::operator==(T)`)
- Хотим явно наложить условия, чтобы облегчить чтение ошибок

# Possible solutions
### Интерфейсы
```cpp
bool contains(const IRange* range, const IVal* value)
```
- Плохое решение - каждый класс придется наследовать. Возможно, будет множественное наследование

### SFINAE
- Проверка на возможность сравнения `T` и `U`
```cpp
template <typename T, typename U, typename = void>
struct is_equality_comparable: std::false_type {};

template <typename T, typename U>
struct is_equality_comparable<T, U, std::void_t<decltype(std::declval<T>() == std::declval<U>())>>: std::true_type {};
```

```cpp
template <typename T, typename U, typename = std::enable_if_t<is_equality_comparable<T, U>::value>>
bool CheckEq(T&& lhs, U&& rhs) {
  return lhs == rhs;
}
```

- Однако ошибка по-прежднему плохо читается
- К тому же, на каждое ограничение надо писать шаблонный аргумент

### `if constexpr` и SFINAE
```cpp
template <typename T, typename U>
bool CheckEq(T&& lhs, U&& rhs) {
  if constexpr (!is_equality_comparable<T, U>::value) {
    static_assert(false, "AAAAA");
  } else {
    return lhs == rhs;
  }
}
```

- Нужна новая ветка ифа на каждое ограничение
- Можем сравнивать не через `==` а через `<` - соответственно опять получим кучу веток ифов

# `requires`
- В `requires` можно передать любое `constexpr-bool` выражение

```cpp
template <typename T, typename U> requires is_equality_comparable<T, U>::value
bool CheckEq(T&& lhs, U&& rhs) {
  return lhs == rhs;
}
```

```cpp
tempate <typename Iter>
requires (std::is_forward_iterator_v<Iter> && std::is_totally_ordered_t<typename Iter::value_type>)
Iter MinElement(Iter first, Iter second) {...}
```

- При этом компилятор будет нам говорить какое именно условие нарушено
- Навешивание `requires` на non-templated функции запрещено
	- _**Note**_ под non-templated подрузумевается отстутствие шаблонов впринципе. Можно навешивать `requires` на методы шаблонного класса

- Syntax
```cpp
requires { requirement-seq }
requires ( parameter-list ﻿(optional) ) { requirement-seq }
```

### Solution
```cpp
template <typename T, typename U>
requires requires(T t, U u) { t == u; }
bool CheckEq(T&& lhs, U&& rhs) {
  return lhs == rhs;
}
```

- `requires`-expression - это `constexpr` предикат над SFINAE характеристиками. Он возвращает `true` если то, что написано внутри него, вообще возможно.
- То есть в нашем случае если `t==u` возможно, то будет `true`
- _**Note**_ Никаких вычислений внутри не происходит!

### Overloading
- По `requires` можно делать перегрузки
```cpp
struct S {
  template <typename Int>
  requires std::is_integral_v<Int>
  S(Int x) {...}

  template <typename Float>
  requires std::is_floating_point_v<Float>
  S(Float x) {...}
}
```

- Проверяется это с точностью до лексем. То есть такие функции считаются разными:
```cpp
template <typename T>
requires (1 == 1)
void f(T x) {}

template <typename T>
requires true
void f(T x) {}
```

- Решили проблему с долгим написанием ограничений
- Пример типичного SFINAE-constaint:
```cpp
template <typename T, typename = void>
struct is_totally_ordered: std::false_type {};

template <typename T>
struct is_totally_ordered<T, std::void_t<
  decltype(std::declval<T>() == std::declval<T>()),
  decltype(std::declval<T>() <= std::declval<T>()),
  decltype(std::declval<T>() < std::declval<T>())
>> : std::true_type {};
```

- Однако по-прежднему не решили проблему с отсутствием определенности - что частное, а что общее
```cpp
template <typename Iter>
requires is_input_iterator_t<Iter>
int distance(Iter first, Iter last) {
  int result = 0;
  while (first != last) {
    ++first;
    ++result;
  }
  return result;
}

template <typename Iter>
requires is_random_iterator<Iter>
int distance(Iter first, Iter last) {
  return last - first;
}
```
- Для RA итератора будут подходить обе функции, потому что наши ограничения не знают что такое частное-общее.

# Concepts
- Не хочется каждый раз писать одинаковый `requires`, если они повторяются
- Формально, концепты - это языковая возможность сочитать разного вида ограничения и запихнуть их в одну сущность
```cpp
template <typename From, typename To>
concept convertible_to = std::is_convertible_v<From, To> && requires {
    static_cast<To>(std::declval<From>());
};

template <typename T>
requires convertible_to<T, int>
int foo() {...}
```
### Концепты из концептов
```cpp
template <typename T, typename U>
concept ComparableWith = requires(T t, U u) {
  {t == u} -> convertible_to<bool>;
  {t != u} -> convertible_to<bool>;
  {u == t} -> convertible_to<bool>;
  {u != t} -> convertible_to<bool>;
};

template <typename T>
concept Comparable = ComparableWith<T, T>;
```

- На сами концепты вешать ограничения нельзя
- А еще концепт считается объявленным только после полного выражения, то есть внутри определения концепта нельзя вызывать его самого
- То есть нет рекурсивного определения

### Сокращенный синтаксис использования
```cpp
// База
template <typename T> requires AmazingConcept<T> void foo(T&);

// Можно писать так
template <AmazingConcept T> void foo(T&);

// Если у концепта несколько шаблонных аргументов
template <AmazingConcept<int, double> T> void foo(T&);
```

# Вложение концептов
- Концепты позволяют нам справится с нашей проблемой "частное и общее"
```cpp
template <typename T> concept Ord = requires(T a, T b) { a < b; };
template <typename T> concept Inc = requires(T a) { ++a; };
template <typename T> concept Void = std::is_same_v<T, void>;

template <typename T> requires Ord<T> || Inc<T> || Void<T>
struct less {
  int operator()() const { return 1; }
}

template <Ord T>
struct less<T> {
  int operator()() const { return 2; }
}

template <>
struct less<void> {
  int operator()() const { return 3; }
}
```
- Вычисления в концптах ленивые, как и в обычных булевых операциях

- В стандарте есть определение отношения subsumes на концептах (вложение):
	- _**Def**_ Констрейнт $P$ включает в себя констрейнт $Q$ если и только если для каждого дизъюнктивного условия $P_i$ в дизъюнктивной нормальной форме $P$ $P_i$ включает в себя каждое конъюктивное условие $Q_j$ в нормальной конъюктивной форме $Q$

- То есть в примере ниже `P` включает в себя `Q` и `R`
```cpp
template <typename T>
concept P = Q<T> && R<T>;
```

- Далее следуют правила, на основе которых выбирается перегрузка
### Rule 1
- Есть `requires` -> более частное
```cpp
template <typename T>
void foo(T) {}

template <typename T>
requires true
void foo(T) {}

int main() {
	foo(0);  // будет вызвана вторая функция
}
```

### Rule 2
- На `requires` не строится ч/о (только на `concept`)
```cpp
template <typename T>
requires ( sizeof<T> == 1; )
void foo(T) {}

template <typename T>
requires ( sizeof<T> == 1; )
void foo(T) {}

int main() {
	foo(0);  // CE: Redefinition - одинаковы полексемно
}
```

```cpp
template <typename T>
requires ( sizeof<T> == 1 && is_same_v<T, int>; )
void foo(T) {}

template <typename T>
requires ( is_same_v<T, int> && sizeof<T> == 1; )
void foo(T) {}

int main() {
	foo(0);  // CE: Ambigious call - проверка с точностью до лексем
}
```

```cpp
template <typename T>
requires ( true; )
void foo(T) {}

template <typename T>
constexpr bool b = true;

template <typename T>
requires ( b<T>; )
void foo(T) {}

int main() {
	foo(0);  // CE: Ambigious call
}
```

### Rule 3
- Overload resolution ordering (`foo(0)`):
	- Сбор overload set
	- Выбров невалидных вызовов (то, что видно явно)
	- Type deduction
	- Temple substitution (подстановка в сигнатуру, SFINAE, etc.)
	- Выбрасываем все `requires "false"`
		- И если в overload set'е осталась хотя бы одна функция с `requires`, то все `non-requires` функции отбрасываются (согласно `rule 1`)
	- Частное/общее
```c
Частное / общее
P subsumes Q (включает в себя )
	=> Q более частное

1) forall P subsumes P
2) P vs Q
	=> Строю КНФ и ДНФ
	если forall P_i из КНФ и forall Q_j и ДНФ -> P_i subsumes Q_j
	// TODO

По итогу получается ЧУМ
И если существует ровно одна перегрузка, которую можно поистине назвать частной над всеми остальными, то выбирается она.
```


# Формы ограничений

Есть 4 формы ограничений:
1. simple requirement
2. type requirement
3. compound requirement
4. nested requirement

```cpp
requires(T t, U u) {
  u + b; // true если u+v возможно (simple)
  typename T::inner; // true если T::inner есть (type)
}
```

### Составные ограничения (compound requirements)
- Составные ограничения проверяют совместимость типов с выражениями
```cpp
requires requires(T x) {
  {*x} -> std::convertible_to<typename T::inner>;
}
```

Это будет `true` если:
1. `*x` валидно
2. `T::inner` валидно
3. `*x` можно сконвиртировать в `T::inner`

- После стрелочки должен идти `type-constraint`
- Концепт `convertible_to` принимает два аргумента, а мы явно указали только 1. Это потому что аргумент автоматически подставится

- Можно проверить на `noexcept` отдельным синтаксисом
```cpp
requires requires(T x) {
  {++x} noexcept;
}
```

### Вложенные ограничения (nested requirements)
- Внутрь `requires` можно засунуть еще `requires`

```cpp
template<class T>
concept Semiregular = DefaultConstructible<T> &&
    CopyConstructible<T> && CopyAssignable<T> && Destructible<T> &&
requires(T a, std::size_t n)
{
    requires Same<T*, decltype(&a)>; // nested: "Same<...> evaluates to true"
    { a.~T() } noexcept; // compound: "a.~T()" is a valid expression that doesn't throw
    requires Same<T*, decltype(new T)>; // nested: "Same<...> evaluates to true"
    requires Same<T*, decltype(new T[n])>; // nested
    { delete new T }; // compound
    { delete new T[n] }; // compound
};
```

_**Note**_ `requires` внутри `requires` - это единственный случай, когда внутри `requires` что-то вычисляется!

- Пример необходимости вычислений `constexpr`-выражения
```cpp
template <typename T>
requires requires {
	sizeof(T) == 4;  // true ВСЕГДА, поскольку элементарно компилируется
}
void foo(T) {}

int main() {
	foo(10);   // OK
	foo(2.2);  // OK
}
```

```cpp
template <typename T>
requires requires {
	requires (sizeof(T) == 4);  // calculates
}
void foo(T) {}

int main() {
	foo(10);   // OK
	foo(2.2);  // CE
}
```
- **Замечание**: выражение внутри nested requires должно быть СТРОГО типа `bool`: компилятор не сможет привести, скажем, `int` к `bool`'у

==TODO==
```cpp
requires (T t) {
	t++;  // can be compiled?
	typename T::value_type;  // defined?
	{++t} -> std::same_as<T&>;
	{++t} noexcept;
	requires (...);
}
```


==TODO== [cppref](https://en.cppreference.com/w/cpp/language/requires)
==TODO== cppstd p.125
==TODO== cppstd p.390
==TODO== семантические и синтаксические требования - см стандарт. Что должно удолвл для каждого концепта

