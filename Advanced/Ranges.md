- Since C++20
# Причина появления
- Хотим исполнять код вида:
```cpp
template <typename InputIt, typename UnaryFunc>
UnaryFunc for_each(InputIt first, InputIt last, UnaryFunc f) {
	for (; first != last; ++first) {
		f(*first);
    }
    return f;
}
```

- При этом `last` - не обязательно `InputIterator`:
```cpp
for_each(istream_iterator<int>{is},
		 istream_iterator<int>{},
		 [os](int d){ if (d < 5) os << d * 2 << " "; }
		);
// last в данном случае даже нельзя двигать!
```

- Также в этой, например, функции мы делаем много действий
```cpp
template <typename InputIt, typename OutputIt,
          typename UnaryPred, typename UnaryFunc>
OutputIt transform_copy_if(
    InputIt first,
    InputIt last,
    OutputIt out_first,
    UnaryPred p,
    UnaryFunc f
);

transform_copy_if(
    istream_iterator<int>{is},
    istream_iterator<int>{},
    ostream_iterator<int>{os, " "},
    [](...){...},
    [](...){...},
)
```

- То есть хочется делать композицию из более мелких заготовок
- Есть вариант такой:
```cpp
std::vector<int> v;
istream_iterator<int> is_begin{is};
istream_iterator<int> is_end{};
ostream_iterator<int> out_begin{os, " "};

std::copy_if(is_begin, is_end, std::back_inserter(v), [](int n){return n < 10; });
std::transform(v.begin(), v.end(), out_begin, [](int n) {return n * 2; });
```
- Проблемы:
	- Копирование
	- Проходимся дважды вместо одного раза
	- То есть около ленивое поведение реализовать не получится

# Ranges
- Solution of previous problem seems like:
```cpp
auto v = std::ranges::istream_view<int>(std::cin)
             | std::ranges::views::filter([](int n){return n < 5; })
             | std::ranges::views::transform([](int n){return n * 2; });

std::ranges::copy(v, std::ostream_iterator<int>{std::cout, " "});
```

_**Def**_ Range - это пара итератор + ограничитель

```cpp
template <typename T> concept range = requires(T&& t) {
    ranges::begin(t);
    ranges::end(t);
};

template <typename T>
using iterator_t = decltype(ranges::begin(declval<T&>()));

template <typename T>
using sentinel_t = decltype(ranges::end(declval<T&>()));
```

- Категории диапазонов определяются иерархически
```cpp
template <typename T> concept input_range = range<T> && input_iterator<iterator_t<T>>;
```
- Остальные ranges определяются аналогичным образом

### `ranges::sort`
```cpp
template<ranges::random_access_range R, class Comp = ranges::less,
         class Proj = std::identity>
requires std::sortable<ranges::iterator_t<R>, Comp, Proj>
constexpr ranges::borrowed_iterator_t<R>
    sort( R&& r, Comp comp = {}, Proj proj = {} );
```

- `random_access_range` - range с категорией random_access
- `Comp` - предикат из библиотеки `ranges`
- Проектор `Proj` - некий функтор, который "проецирует" объект. Например можно использовать указатель на член класса `&Class::field`

```cpp
ranges::sort(v, [](Struct a, Struct b) {return a.x < b.x; });
ranges::sort(v, {}, [](Struct a) {return a.x; });
ranges::sort(v, {}, &Struct::x);
```

- Встает вопрос, что возвращать:
	1. Ничего (как `std::sort`)
	2. Отсортированный range
	3. Итератор на конец отсортированного диапозона
- На самом деле, работая с ranges, хочется возвращать именно что итератор на конец отсортированного диапозона, чтобы например делать такое:
	- `ranges::unique(ranges::begin(vec), ranges::sort(vec));`
- Но есть проблема: итератор может "провиснуть"
```cpp
auto it = ranges::sort(foo());
auto elem = *it; // все сломалось
```
- Стоит ввести специальную обертку

### `borrowed_iterator`
- `borrowed_iterator_t` это обертка: содержит либо итератор, либо `ranges::dangling` (который просто нельзя разыменовать)
```cpp
template<typename R>
concept borrowed_range = ranges::range<R> &&
    (std::is_lvalue_reference_v<R> ||
     ranges::enable_borrowed_range<std::remove_cvref_t<R>>);
```

- То есть `dangling` будет возвращаться, если это rvalue и не включен `enable_borrowed_range` для этого класса
- `enable_borrowed_range` можно выставить для своего класса, в STL он выставлен в `true` например для `std::basic_string_view` (не может провиснуть даже если rvalue)

- Possible implementation of `borrowed_iterator_t`
```cpp
template<std::ranges::range R>
using borrowed_iterator_t = std::conditional_t<
	std::ranges::borrowed_range<R>,
	std::ranges::iterator_t<R>,
	std::ranges::dangling
>;
```

# `views`
```cpp
template <typename T> concept view = range<T> && movable<T> && enable_view<T>;
template <typename T> constexpr bool enable_view = derived_from<T, view_base>;
```

- Интуитивно: отображение это не владеющий диапазон
- Что-то похожее уже было: `string_view` (владеет `char*` и `size*`)
- Проблемы:
	- Не знает, на что он `view` (всегда `view` просто на `char*`)
	- Не может вернуть объект, на который он `view`
- Альтернатива: `rev_view`

### `rev_view`
- Предоставляет вью на объект, который под ним лежит
```cpp
auto sv = views::all(s); // -> ranges::ref_view<string>
auto s = sv.base(); // std::string
```

- При этом требует от владельца только наличие `begin` и `end`:
```cpp
struct S {
    int* begin() { return nullptr; }
    int* end() { return nullptr; }
};

S s;
auto sv = views::all(s);
```

# `sentinel`
```cpp
#include <iostream>
#include <ranges>
#include <vector>

int accumulate(const std::ranges::input_range auto& coll) {
  int acc = 0;
  for (auto& elem : coll) acc += elem;
  return acc;
}

template <int Value>
struct EndValue {
  template <typename It>
  bool operator==(It it) const {
    return *it == Value;
  }
};

int main() {
  std::vector<int> v{1, 2, 3, 4};
  auto w = std::ranges::subrange(v.begin() + 1, EndValue<4>{});
  auto res = accumulate(w);
  std::cout << res;  // 2 + 3 = 5
}
```

# `subrange`
- `subrange` позволяет нам взять `range` на подпоследовательность элементов. Но это опять просто пара итераторов.
```cpp
std::multimap<int, char> m;
auto [first, last] = m.equal_range(k);
for (auto& [k, v] : ranges.subrange(first, last)) {
    ...
}
```

- Напишем числа Фибоначчи в виде итератора
```cpp
class FibIter {
 public:
  int operator*() const {return cur_; }

  FibIter& operator++() {
    cur_ = std::exchange(next_, cur_ + next_);
    return *this;
  }
 private:
  int cur_ = 0;
  int next_ = 1;
};
```

- Пользоваться этим не очень удобно, воспользуемся счетным итератором:
```cpp
FibIter fi;
auto fib_i = std::counted_iterator{fi, N}
for (auto it = fib_i; it != std::default_sentinel; ++it) {
  // ...
}
```

- Сделаем счетный `subrange`
- Для начала введем понятие счетного ренжа
```cpp
template<typename T>
concept sized_range = ranges::range<T> &&
                      requires(T& t) { ranges::size(t); };
```

- Из счетного итератора и ограничителя уже можно сделать `subrange`:
```cpp
FibIter fi;
auto fib_i = std::counted_iterator{fi, N};
auto fib_sub = ranges::subrange{fib_i, std::default_sentinel};
```

_**Note**_ `default_sentinel` - это такой ограничель, который перекладывает ответственность за проверку конца на сам итератор

- А еще можно сделать бесконечный `subrange`
```cpp
auto fib_view = ranges::subrange<FibIter, unreachable_sentinel_t>{};
std::vector<int> v(10);
ranges::copy_n(fib_view.begin(), 10, v.begin());
```

- Можно создавать `subrange` через `take`:
```cpp
std::string s = "Hello";
auto sv = views::all(s); // -> ref_view<string>
auto tk = views::take(sv, 5); // -> take_view<ref_view<string>>
auto fv = ranges::subrange{fib_iter, std::unreachable_sentinel};
auto fib_tk = views::take(fv, 10);
```

# `common_view`
```cpp
FibIter fi;
auto fib_i = std::counted_iterator{fi, N};
auto fib_sub = ranges::subrange{fib_i, std::default_sentinel};
```
- С таким `subrange` не будет работать `std::sort`, потому что `std::sort` требует совпадения типа итераторов (`begin` и `end`)

- Поэтому есть хак: `common_view`
```cpp
template<typename T>
concept common_range = ranges::range<T> &&
                       std::same_as<
	                       ranges::iterator_t<T>,
	                       ranges::sentinel_t<T>
	                    >;

auto c1 = std::ranges::common_view{r1};
```
- `c1` уже можно передавать в стандартные алгоритмы

# Комбинация отображений
```cpp
auto v = ranges::views::transform(
  ranges::views::filter(
    ranges::istream_view<int>(is),
    [](int n){ return n < 5; }),
  [](int n){ return n * 2; });
```

- Читать это неудобно, поэтому проще писать так:
```cpp
auto v = ranges::istream_view<int>(is)
        | ranges::views::filter([](int n){ return n < 5; })
        | ranges::views::transform([](int n){ return n * 2; });
```

# Устройство `view` на примере `transform_view`
- Давайте хранить функтор во вью
```cpp
template<input_range Range, copy_constructible Fn>
class transform_view : public view_interface<transform_view<Range, Fn>> {
  struct Iter {
    Fn f;
    iterator_t<Range> baseit;
	// ...
    decltype(auto) operator*() const {
      return f(*baseit);
    }
  };
  // ...
};
```

- Это плохо, потому что `Fn` может весить много, поэтому в libstdc++ решили хранить указатель на `parent`:
```cpp
template<input_range Range, copy_constructible Fn>
class transform_view : public view_interface<transform_view<Range, Fn>> {
  Fn f;
  struct Iter {
    transform_view *parent;
    iterator_t<Range> baseit;
	// ...
    decltype(auto) operator*() const {
      return parent->f(*baseit);
    }
  };
  // ...
};
```

# Range factories

## `std::ranges::all`
- Самое главное - `std::ranges::all(r)`
- Перегружно по концептам:
	1) Случай I. `r` уже view -> `return r;`
	2) Случай II. `r` - это l-value range -> `return reference_view(r);`
	3) Случай III. `r` - это r-value range -> `return owning_view(r);`

Поэтому при `vec | filter` `operator|` возвращает `views::all` от `filter`

## single
- Всего один элемент
- `single(10)`
- Владеющий view

## empty
- Всегда пустой
- `empty()`

## iota
- `iota(1, 10)` - цикл от 1 до 10
- `iota(1)` - бесконечная ленивая последовательность от 1
- `iota(2) | filter(isPrime) | take(10)` - первые 10 простых чисел

## repeat
- `repeat(3)` - постоянно возвращает `3`

# Examples

### Stride view
```cpp
#include <bits/iterator_concepts.h>
#include <concepts>
#include <iostream>
#include <iterator>
#include <ranges>
#include <vector>

template <typename T>
concept Range = requires (T t) {
  // t.begin();
  // t.end();
  std::ranges::begin(t);
  std::ranges::end(t);
};  // O*(1)
// [begin, end)

template <typename T>
using iterator_t = decltype(std::ranges::begin(std::declval<T>()));

template <typename T>
using sentinel_t = decltype(std::ranges::end(std::declval<T>()));

// Ranges in std
// ALL stl containers
// string
// string_view
// span (like a string_view, but for vector)
// C static arrays
// C++26 optional

template <typename T>
concept EnableView = std::derived_from<T, std::ranges::view_base> ||
  std::derived_from<T, std::ranges::view_interface<T>>;

template <typename T>
concept View = Range<T> && std::movable<T> && EnableView<T>;



template <std::ranges::range R>
class StrideView : std::ranges::view_interface<StrideView<R>> {
  struct Iter {
    using difference_type = std::ptrdiff_t;
    using iterator_category = std::forward_iterator_tag;
    Iter(std::ranges::iterator_t<R> it,
         std::ranges::sentinel_t<R> end,
         std::size_t stride) : it_(it), end_(end), stride_(stride) {}

    auto& operator*() { return *it_; }

    Iter& operator++() {
      for (std::size_t i = 0; i < stride_; ++i) {
        if (it_ == end_) {
          break;
        }
        ++it_;
      }
      return *this;
    }

    Iter operator++(int) {
      Iter copy = *this;
      for (std::size_t i = 0; i < stride_; ++i) {
        if (it_ == end_) {
          break;
        }
        ++it_;
      }
      return copy;
    }

    bool operator==(std::default_sentinel_t) const {
      return it_ == end_;
    }

    std::ranges::iterator_t<R> it_;
    std::ranges::sentinel_t<R> end_;
    std::size_t stride_;
  };

 public:
  using value_type = typename R::value_type;
  using reference = typename R::reference;
  using iterator = Iter;

  StrideView(R& range, std::size_t stride)
    : stride_(stride),
      begin_(std::ranges::begin(range)),
      end_(std::ranges::end(range)) {
  } 

  auto begin() { return Iter(begin_, end_, stride_); }
  auto end() { return std::default_sentinel; }

  std::size_t stride_;
  std::ranges::iterator_t<R> begin_;
  std::ranges::sentinel_t<R> end_;
};

template <typename T>
void Print(T&) {
  std::cout << __PRETTY_FUNCTION__ << '\n';
}

int main() {
  std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
  for (auto& elem : vec) {
    std::cout << elem << ' ';
  }
  std::cout << "\n";

  auto s1 = StrideView(vec, 2);
  for (auto& elem : s1) {
    std::cout << elem << ' ';
  }
  std::cout << '\n';

  auto s2 = StrideView(s1, 3);
  for (auto& elem : s2) {
    std::cout << elem << ' ';
  }
  // 1 3
  std::cout << "\n";

  Print(s2);
}
```

### Filter view
```cpp
#include <cstddef>
#include <iostream>
#include <iterator>
#include <ranges>
#include <type_traits>
#include <utility>
#include <vector>

template <std::ranges::range R, typename F>
class FilterView : std::ranges::view_interface<FilterView<R, F>> {
 private:
  using base_iterator_t = std::ranges::iterator_t<R>;
  using base_sentinel_t = std::ranges::sentinel_t<R>;

  struct Iterator {
    base_iterator_t iter_;
    base_sentinel_t sent_;
    FilterView* base_;

    using difference_type = std::ptrdiff_t;
    using value_type = std::iter_value_t<base_iterator_t>;
    using reference = std::iter_reference_t<base_iterator_t>;
    using iterator_category = std::forward_iterator_tag;
    using iterator_concept = std::forward_iterator_tag;

    Iterator(FilterView* ptr, base_iterator_t iter, base_sentinel_t sent) :
        base_(ptr), iter_(iter), sent_(sent) {}

    Iterator& operator++() {
      ++iter_;
      while (iter_ != sent_ && not base_->predicate_(*iter_)) {
        ++iter_;
      }
      return *this;
    }

    Iterator operator++(int) {
      auto copy = *this;
      ++(*this);
      return copy;
    }

    reference operator*() const {
      return *iter_;
    }

    bool operator==(const std::default_sentinel_t& sent) const {
      return iter_ == sent_; 
    }

  };

 public:
  FilterView() = default;
  
  template <std::ranges::range R2, typename F2>
  FilterView(R2&& range, F2&& predicate)
    : base_(std::forward<R2>(range)), iter_(this, std::ranges::begin(base_), std::ranges::end(base_)), predicate_(std::forward<F2>(predicate)) {}

  Iterator begin() {
    if (cached_) {
      return iter_;
    }
    cached_ = true;
    if (iter_ != end() && not predicate_(*iter_)) {
      ++iter_;
    }
    return iter_;
  }
  auto end() const { return std::default_sentinel_t{}; }

 private:
  R base_;
  bool cached_ = false;
  Iterator iter_;
  F predicate_;
};

template <std::ranges::range R2, typename F2>
FilterView(R2&& range, F2&& predicate) -> FilterView<std::decay_t<R2>, std::decay_t<F2>>;


template <typename T>
void Print(T& container) {
  for (auto elem : container) {
    std::cout << elem << '\n';
  }
  std::cout << "\n\n";
}

template <std::size_t M>
class Predicate {
 public:
  int y = 0;
  int z = 10;
  bool operator()(auto x) {
    ++y;
    std::cout << "y = " << y << z << '\n'; 
    return x % M == 0;
  }
};

int main() {
  std::vector v = {1, 2, 3, 4, 5, 6, 7, 8};
  auto filtered = FilterView(v, Predicate<2>{});
  auto filtered2 = FilterView(
      FilterView(v, Predicate<2>{}),
      Predicate<3>{});
  Print(filtered2);
}

```

# About pipe
- `|` - operator pipe

```cpp
// Old syntax
take(transform(filter(vec, pred), pr), 10)

// New syntax
filter(v, pred) | transform(pr) | take(10)
```

- Проблема разработки состоит в том, чтобы инвертировать views
- Вводим `ViewAdapter`, который позволит писать 4 вариации объявления `FilterView`:
1) `FilterView(v, pr)`
2) `Filter(range, pred)`
3) `Filter(pred)(range)`
4) `range | Filter(pred)`

```cpp
template <R, P>
Filter(R r, P p) {
	return FilterView(r, p);
}

template <P>
Filter(P p) {
	return [P p /* perfect forwarding */](auto range) { return FilterView(r, p); }
	// Вообще FilterClosure - то, что хранит предикат и располагает operator()
}

template <R>
auto operator|(R r, FilterClosure<P> c) {
	return c(r);
	// Вообще c(std::ranges::all(r));
}
```
==TODO== checkout pipe
