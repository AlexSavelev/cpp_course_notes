# Example
`std::vector<int>::iterator it = v.begin();`
- In STL: `vector<T>`: `using iterator = ...;`

# Range-based for
`for (auto x : v) { ... }`
- Equivalent to: `for (it = ...) { auto x = *it; ... }`
	- `*it` - unnaming
- Always write `auto`
	- Better: `const auto&` or `auto&` (if we want to change it)

# Iterators categories

| Iterator category           | write | read | inc (no multi-pass) | inc (multi-pass) | dec | RA  | contiguous storage |
| --------------------------- | ----- | ---- | ------------------- | ---------------- | --- | --- | ------------------ |
| LegacyIterator              |       |      | +                   |                  |     |     |                    |
| LegacyOutputIterator        | +     |      | +                   |                  |     |     |                    |
| LegacyInputIterator         |       | +    | +                   |                  |     |     |                    |
| LegacyForwardItetator       |       | +    | +                   | +                |     |     |                    |
| LegacyBidirectionalIterator |       | +    | +                   | +                | +   |     |                    |
| LegacyRandomAccessIterator  |       | +    | +                   | +                | +   | +   |                    |
| LegacyContiguousIterator    |       | +    | +                   | +                | +   | +   | +                  |

- Input (`++`, `*`, `==`, (from C++20 `!=`)) (`std::istream`)
	- Forward (целостность данных, т.е. можно проходиться по несколько раз) (`forward_list`)
	- Bidirectional (`--`)
	- Random Access (`+=`, `-=`, `+`, `-`)
	- Contiguous (from C++20) (данные в памяти непрерывны) (`vector`, `array`, `string`)
		- Полезно, например, для `std::copy` (-> `memcpy` + `constructors()`)
- Output
	- `back_inserter(vec)`
	- `*it = 3;  // push back to vector`
	- `std::ostream`

# `std::iterator_traits<Iter>`
- Это специальная структура данных, которая позволяет унифицированно работать с итераторами
- Внутри этой структуры есть следующие юзинги:
	1. difference_type
	2. value_type
	3. pointer
	4. reference
	5. iterator_category
- `typename std::iterator_traits<Iter>::value_type val = *it;`

- `iterator_traits` определяет эти поля следующим образом:
	- До C++20: если в `Iterator` нет хотя бы одного из 5 юзингов то `std::iterator_traits` пустая. Иначе все юзинги равны соответствующим юзингам из `Iterator`.
	- Начиная с C++20: если в Iterator нет юзинга pointer (но есть все остальное), то pointer = void, остальное по той же схеме.
_**Note**_ `iterator_traits` можно специализировать для своего класса

# `std::advance`
- Эта функция позволяет "продвинуть" итератор на n шагов
- `std::advance(it, 3);`

### Realization - version 1
- RA итератор будем двигать сразу, не RA - постепенно
```cpp
#include <iterator>
#include <type_traits>

template <typename Iter>
void Advance(Iter& it, int n) {
  if (std::is_same_v<typename std::iterator_traits<Iter>::iterator_category,
                     std::random_access_iterator_tag>) {
    it += n;
  } else {
    for (int i = 0; i < n; ++i) {
      ++it;
    }
  }
}
```

- Problem: CE (we don't know about `if constexpr` (since C++17))
### Realization - version 2
```cpp
#include <iterator>

template <typename Iter, typename IterCategory>
void Helper(Iter& it, int n, IterCategory) {
  for (int i = 0; i < n; ++i) {
    ++it;
  }
}

template <typename Iter>
void Helper(Iter& it, int n, std::random_access_iterator_tag) {
  it += n;
}

template <typename Iter>
void Advance(Iter& it, int n) {
  Helper(it, n, std::iterator_traits<Iter>::iterator_category());
}
```

# Const iterators
- `vector<int>::const_iterator`
- `v.cbegin();`

# Reverse iterators
- `vector<int>::reverse_iterator`
- `v.rbegin();`

# `std::copy`, `std::copy_if`
- `std::copy(begin, end, out_begin)`

# Output iterators & adapters

### Problem
```cpp
std::list<int> l = {1, 2, 3, 4};
std::vector<int> v;
std::copy(l.begin(), l.end(), v.begin());
```
- Код не валиден, поскольку `std::copy` не проверяет на размерность `dest`

### Solution
```cpp
std::list<int> l = {1, 2, 3, 4};
std::vector<int> v;
std::copy(l.begin(), l.end(), std::back_inserter(v));
```

### `back_inserter` realization
- `back_inserter` - адаптер

```cpp
template <typename Container>
class back_insert_iterator {
 public:
  back_insert_iterator(Container& container) : container(container) {}

  back_insert_iterator<Container>& operator++() { return *this; }

  back_insert_iterator<Container>& operator*() { return *this; }

  back_insert_iterator<Container>& operator=(
      const typename Container::value_type& value) {
    container.push_back(value);
    return *this;
  }

 private:
  Container& container;
};

template <typename Container>
back_insert_iterator<Container> back_inserter(Container& container) {
  return back_insert_iterator<Container>(container);
}
```

- Существуют `front_insert_iterator` (`push_front`) и просто `insert_iterator` (`insert`) (для соответствующих контейнеров)

# Stream итераторы
- Пример: итератор на `std::cin`
```cpp
std::istream_iterator<int> it(std::cin);
for (; it != std::istream_iterator<int>(); ++it) {
    std::cout << *it;
    ++it;
}
```

### Implementation
```cpp
template <typename T>
class istream_iterator {
 public:
  istream_iterator(std::istream& in): in_(in) {
    in_ >> value;
  }

  istream_iterator<T>& operator++() {
    in_ >> value;
  }

  T& opeartor*() {
    return value;
  }

 private:
  T value;
  std::istream& in_;
}
```

