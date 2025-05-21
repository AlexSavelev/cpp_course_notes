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
	- Около ленивое поведение реализовать не получится

# Ranges
- Solution of prev problem seem like:
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

- Категории диапозонов определяются иерархически
```cpp
template <typename T> concept input_range = range<T> && input_iterator<iterator_t<T>>;
```

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
- Проектор - некий функтор, который "проецирует" объект. Например можно использовать указатель на член класса `&Class::field`

```cpp
ranges::sort(v, [](Struct a, Struct b) {return a.x < b.x; });
ranges::sort(v, {}, [](Struct a) {return a.x; });
ranges::sort(v, {}, &Struct::x);
```

### `borrowed_iterator`
- `borrowed_iterator_t` это обертка: содержит либо итератор, либо `ranges::dangling` (который просто нельзя разыменовать)
```cpp
template<typename R>
concept borrowed_range = ranges::range<R> &&
    (std::is_lvalue_reference_v<R> ||
     ranges::enable_borrowed_range<std::remove_cvref_t<R>>);
```

- То есть `dangling` будет возвращаться если это rvalue и не включен `enable_borrowed_range` для этого класса
- `enable_borrowed_range` можно выставить для своего класса, в STL он выставлен в `true` например для `std::basic_string_view` (не может провиснуть даже если rvalue)

# `views`
```cpp
template <typename T> concept view = range<T> && movable<T> && enable_view<T>;
template <typename T> constexpr bool enable_view = derived_from<T, view_base>;
```

- Интуитивно: отображение это не владеющий диапозон
- Что-то похожее уже было: `string_view` (владеет `char*` и `size*`)
- Проблемы:
	- Не знает на что он `view` (всегда `view` просто на `char*`)
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
int accumulate(const ranges::input_range auto& coll) {
    int acc = 0; for (auto &elem : coll) acc += elem;
    return acc;
}

template <int Value>
struct EndValue {
    template <typename It>
    bool operator==(It it) const {
        return *it == Value;
    }
}

std::vector<int> v{1, 2, 3, 4};
auto w = ranges::subrange(v.begin() + 1, EndValue<4>{});
auto res = accumulate(w);
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
  int next_ = 0;
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
                       std::same_as<ranges::iterator_t<T>, ranges::sentinel_t<T>>;
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

==TODO== sem26 code listings

# With pipe
- `|` - operator pipe ==TODO==
- ==TODO== кложуры (closure)
- ==TODO== `enable_borrowed_range`

```cpp
take(transform(filter(vec, pred), pr), 10)

// New syntax
filter(v, pred) | transform(pr) | take(10)
```

- Цель: инвертировать views
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
