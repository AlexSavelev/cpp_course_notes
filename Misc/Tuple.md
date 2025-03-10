# Implementation 1
- Невзирая на рекурсию в коде, компилятор все заинлайнит и все будет работать за $O(1)$

```cpp
#include <iostream>

template <typename... Ts>
class Tuple;

template <typename Head, typename... Tail>
class Tuple<Head, Tail...> {
 public:
  Tuple(Head head, Tail... args) : value(head), tail(args...) {}

  Head value;
  Tuple<Tail...> tail;
};

template <>
class Tuple<> {
 public:
  Tuple() = default;
};

template <typename Tup>
struct TupleSize {};

/*
template <typename Head, typename... Tail>
struct TupleSize<Tuple<Head, Tail...>> {
  static const std::size_t value =
      1 +
      TupleSize<Tuple<Tail...>>::value;  // const = constexpr for integral types
};
*/ // in cringe.cpp

template <typename... Ts>
struct TupleSize<Tuple<Ts...>> {
  static const std::size_t value =
      sizeof...(Ts);  // O(1) or O(n) - implementation defined??
};

template <>
struct TupleSize<Tuple<>> {
  static const std::size_t value = 0;
};

using TestTup1 = Tuple<int, double, float>;
using TestTup2 = Tuple<int, int>;

static_assert(TupleSize<TestTup1>::value == 3);
static_assert(TupleSize<TestTup2>::value == 2);
static_assert(TupleSize<Tuple<>>::value == 0);

// Getting element
template <std::size_t I, typename Tup>
struct TupleElement {
  static_assert(0, "Wrong Type Index");
};

template <std::size_t I, typename Head, typename... Tail>
struct TupleElement<I, Tuple<Head, Tail...>> {
  using type = typename TupleElement<I - 1, Tuple<Tail...>>::type;
};

template <typename Head, typename... Tail>
struct TupleElement<0, Tuple<Head, Tail...>> {
  using type = Head;
};

template <std::size_t I, typename Tup>
using TupleElementT = typename TupleElement<I, Tup>::type;

// Getting element
template <std::size_t I, typename Tup>
struct GetImpl {};

template <std::size_t I, typename Head, typename... Tail>
struct GetImpl<I, Tuple<Head, Tail...>> {
  static TupleElementT<I, Tuple<Head, Tail...>>& get(
      Tuple<Head, Tail...>& tuple) {
    return GetImpl<I - 1, Tuple<Tail...>>::get(tuple.tail);
  }
};

template <typename Head, typename... Tail>
struct GetImpl<0, Tuple<Head, Tail...>> {
  static TupleElementT<0, Tuple<Head, Tail...>>& get(
      Tuple<Head, Tail...>& tuple) {
    return tuple.value;
  }
};

template <std::size_t I, typename Tup>
TupleElementT<I, Tup>& get(Tup& tuple) {
  return GetImpl<I, Tup>::get(tuple);
}

/* Partial spec. is not exist for function because of overloading
template <typename Tup>
TupleElementT<0, Tup>& get(Tup& tuple) {
  return tuple.value_;
}
*/

int main() {
  TestTup1 tup(1, 2.2, 3.3f);
  std::cout << get<0>(tup) << '\n';
  std::cout << get<1>(tup) << '\n';
  get<1>(tup) = 5.5;
  std::cout << get<1>(tup) << '\n';
  std::cout << get<2>(tup) << '\n';
  return 0;
}
```


# Implementation 2
```cpp
#include <iostream>
#include <utility>

template <std::size_t I, typename T>
struct ValueHolder {
  T value;
};

template <typename T, typename... Ts>
struct TupleBase;

template <std::size_t... Is, typename... Ts>
struct TupleBase<std::index_sequence<Is...>, Ts...> : ValueHolder<Is, Ts>... {
  TupleBase(Ts... args) : ValueHolder<Is, Ts>(args)... {}
};

// template <std::size_t... Is> IndexSequence {};  // cool class

template <typename... Ts>
struct Tuple : TupleBase<std::make_index_sequence<sizeof...(Ts)>, Ts...> {
  Tuple(Ts... args)
      : TupleBase<std::make_index_sequence<sizeof...(Ts)>, Ts...>(args...) {}
};

using TestTuple1 = Tuple<int, double, float>;
// intristic

template <std::size_t I, typename T>
auto get(ValueHolder<I, T>& holder) {
  return holder.value;
}

template <typename T, std::size_t I>
auto get(ValueHolder<I, T>& holder) {
  return holder.value;
}

/* Can to do
// make TupleCatImpl: return Tuple<..., ...>(get(IL)..., get(IR)...);
// where IL, IR is std::make_index_seq<sizeof...(IL)

template <typename... TL, typename... TR>
Tuple<TL..., TR...> TupleCat(Tuple<TL...> lhs, Tuple<TR...> rhs) {}
*/

int main() {
  TestTuple1 tup(1, 2.2, 3.3);
  std::cout << get<0>(tup) << '\n';
  std::cout << get<1>(tup) << '\n';
  std::cout << get<2>(tup) << '\n';
  std::cout << get<int>(tup) << '\n';
}
```

