# `std::vector`
- _**Def:**_ Последовательный набор элементов
### Доступ к элементам
- `at`/`[]`: возвращает элемент по индексу (в первом случае выполняется проверка индекса). Работает за $O(1)$
- `front`/`back`. Возвращает первый/последний элемент. В случае пустого вектора - UB. Работает за $O(1)$
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(n)$ (вынуждает весь вектор подвинуться)
- `erase`: удаление по итератору: $O(n)$
- `push_back`/`emplace_back`: вставка в конец. Работает амортизированно за $O(1)$
- `pop_back`: удаление с конца. Работает за $O(1)$. Вызов от пустого вектора: UB
### Итератор
- Категория: Contiguous

# `std::deque`
- _**Def:**_ Набор элементов, который поддерживает вставку как в конец так и в начало. Хранит элементы в "бакетах"
### Доступ к элементам
- `at`/`[]`: возвращает элемент по индексу (в первом случае выполняется проверка индекса). Работает за $O(1)$
- `front`/`back`. Возвращает первый/последний элемент. В случае пустого дека - UB. Работает за $O(1)$
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(n)$
- `erase`: удаление по итератору: $O(n)$
- `push_back`/`emplace_back`/`push_front`/`emplace_front`: вставка в конец/начало. Работает амортизированно за $O(1)$
- `pop_back`/`pop_front`: удаление с конца/начала. Работает за $O(1)$. Вызов от пустого дека: UB
### Итератор
- Категория: RandomAccess

# `std::list`
- _**Def:**_ Обычный двусвязный список
- `Node { val, *next, *prev }`
- Есть фейковая нода (`begin.prev`, `end`)
	- С этим допускается следующее:

```cpp
auto t = l.begin();
l.push_back(1);
l.push_back(2);
*(t - 2); // 1
```

_**Sol**_: создаем класс `BaseNode { *next, *prev }`, а `Node` - наследник `BaseNode` с `T value`
- По умолчанию `next` и `prev` у фейковой ноды - ссылки на саму себя
### Доступ к элементам
- `front`/`back`. Возвращает первый/последний элемент. В случае пустого листа - UB. Работает за $O(1)$
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(1)$
- `erase`: удаление по итератору: $O(1)$
- `push_back`/`emplace_back`/`push_front`/`emplace_front`: вставка в конец/начало. Работает за $O(1)$
- `pop_back`/`pop_front`: удаление с конца/начала. Работает за $O(1)$
### Итератор
- Категория: BidirectionalIterator

# `std::forward_list`
- _**Def:**_ Обычный односвязный список
### Доступ к элементам
- `front`. Возвращает первый элемент. В случае пустого листа - UB. Работает за $O(1)$
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(1)$
- `erase`: удаление по итератору: $O(1)$
- `push_front`/`emplace_front`: вставка в начало. Работает за $O(1)$
- `pop_front`: удаление с начала. Работает за $O(1)$
### Итератор
- Категория: ForwardIterator

# `std::map`
- _**Def:**_ Отсортированный по ключу ассоциативный контейнер который хранит пары ключ-значение. Внутри обычно используется красно-черное дерево поиска.
- `Node { kv, left, right, parent, is_red }`
	- `kv` -> `std::pair<const Key, Value> kv;`
- Итератор к пустой `map`'е также можно реализовать через фейковую ноду - "самую правую"
- _**Sol**_: создаем класс `BaseNode { *left, *right, *parent }`, а `Node` - наследник `BaseNode` с `kv` и `is_red`
- _**Note**_: Поскольку имеем дело с `const Key`, лучше в `range-base for`'е использовать `auto`
### Доступ к элементам
- `index[]` - доступ по ключу. Модифицрует контейнер - если значения с таким ключом не существует то создает его (кладет туда `Value()`). Работает за $O(logn)$
- `at` - доступ по ключу. Если значения с таким ключом не существует то выкидывает ошибку. Работает за $O(logn)$
- `find` - возвращает итератор на пару с нужным ключем. Если ключа нет возвращает `end()`. Работает за $O(logn)$
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(logn)$
- `erase`: удаление по итератору: $O(logn)$
### Итератор
- Категория: BidirectionalIterator

# `std::set`
- _**Def:**_ Множество элементов. По сути это map без value

# `std::unordered_map`
- _**Def:**_ Ассоциативный контейнер который хранит пары ключ-значение. Является хэш-таблицей
### Доступ к элементам
- `index[]` - доступ по ключу. Модифицрует контейнер - если значения с таким ключом не существует то создает его (кладет туда `Value()`). Работает за $O(1)$ (не совсем честно).
- `at` - доступ по ключу. Если значения с таким ключом не существует то выкидывает ошибку. Работает за $O(1)$ (не совсем честно).
- `find` - возвращает итератор на пару с нужным ключем. Если ключа нет возвращает `end()`. Работает за $O(1)$ (не совсем честно).
### Модифицирующие операции
- `insert`/`emplace`: вставка по итератору. Работает за $O(1)$ (не совсем честно).
- `erase`: удаление по итератору: Работает за $O(1)$ (не совсем честно).
### Итератор
- Категория: ForwardIterator

# `std::unordered_set`
- _**Def:**_ Неупорядоченное множество элементов. По сути это `unordered_map` без `value`

# `std::multimap`, `std::multiset`
- _**Def:**_ `std::unordered_map/set` с поддержкой одинаковых ключей

# Adapters
- В STL существуют так же адаптеры над контейнерами: `stack`, `queue`, `priority_queue`. Работают они довольно просто: используют методы контейнера который хранят в полях. Например, `std::stack` использует `std::deque` по умолчанию

# Инвалидация итераторов
- Доступна [здесь](https://en.cppreference.com/w/cpp/container)
- К экзу лучше выучить
==TODO== party 05

# String
### std::string
- gcc realization (`data*`, `size`, `padding`, `capacity`)
- Смешная история:
	- `if (data[size_] != 0) { data[size_] = '\0'; }`
	- Это UB: read of unitialized data
	- Причем работает на 192 символах (или что-то типа того) станица памяти, физическая страница ==TODO== what is it

### SSO implementation
- _**SSO**_ - is the Short / Small String Optimization
- [See more](https://stackoverflow.com/questions/10315041/meaning-of-acronym-sso-in-the-context-of-stdstring)
```cpp
class string {
public:
    // all 83 member functions
private:
    size_type m_size;
    union {
        class {
            // This is probably better designed as an array-like class
            std::unique_ptr<char[]> m_data;
            size_type m_capacity;
        } m_large;
        std::array<char, sizeof(m_large)> m_small;
    };
};
```

#### SSO benchmark on Ubuntu
![SSO benchmark](../assets/SSO_benchmark.png)
#### std::string SSO
[Source](https://stackoverflow.com/questions/27631065/why-does-libcs-implementation-of-stdstring-take-up-3x-memory-as-libstdc/28003328#28003328)

| Compiler         | Stack space (bytes) | Heap space (bytes) | Capacity (char8_t / bytes) |
| ---------------- | ------------------- | ------------------ | -------------------------- |
| **gcc 4.9.1**    | 8                   | 27                 | 2                          |
| **gcc 5.0.0**    | 32                  | 0                  | 15                         |
| **clang/libc++** | 24                  | 0                  | 22                         |
| **VS-2015**      | 32                  | 0                  | 15                         |

### COW
- _**COW**_ - Copy-on-write
- Подход, при котором во время чтения области данных используется общая копия, а в случае изменения данных — создается новая копия.

### FBString
- Фейсбуковская (folly FBString)
- fields order: `data`, `capacity`, `size`
- В последнем байте лежит оставшийся `capacity`
	- `Hello world!abcdefjhij\0` (`\0` все равно детерминирующий ноль)
- На самом деле не в последнем байте, а в последних 5 битах, остальные 3 бита уходят под флаги
	- `SSO/NO SSO`
- В `STL` реализация не была принята ввиду необходимости постоянно обращаться к `size` и срезать биты при обращении к `data[i]`
- Инструкции в зависимости от размера `S`:

| S <= 23 | 23 < S < 256 | S >= 256 |
| ------- | ------------ | -------- |
| SSO     | as vector    | COW      |

# `std::tuple`

```cpp
std::tuple<int, double, std::string> tup(1, 2.2, "Hello");
```

### Get items
```cpp
std::get<0>(tup);  // get by index
std::get<double>(tup);  // get by type (get in tuple with two doubles is CE)
```

#### Cringe get
```cpp
struct Tuple {
	std::string z;
	double y;
	int x;
};

int main() {
	std::tuple<int, double, std::string> tup(1, 2.2, "Hello");
	std::cout << reinterpret_cast<Tuple&>(tup).x;  // 1 but UB
}
```

### Create tuple and unpack values
- `auto tup = std::make_tuple(0, 1.1, "world");`

```cpp
auto tup = std::make_tuple(0, 1.1, "world");  // T = int
std::tuple tup1 = {1, 2.2, "Hello"};  // deduction guide (from C++17)

int x = 37;
auto tup = std::make_tuple(x, 1.1, "world");  // T = int

// unpack
auto [x, y, z] = tup;

// std::tie - makes tuple by Args&...
int a = 37;
double b = 1.1;
std::string c = "lala";
std::tie(a, b, c) = tup;  // makes 3-linked tuple and assigns to tup
std::cout << x << ' ' << y << ' ' << z << '\n';
```

### Tuple cat
- `std::tuple tup2 = std::tuple_cat(tup, tup1);`

### `std::forward_as_tuple`
### Каррирование
- ==TODO== Perfect forwarding
```cpp
#include <iostream>
#include <tuple>

template <typename F, typename... Args>
auto curry(F&& f, Args&& args) {
	return [
		ft = std::forward_as_tuple(std::forward<F>(f)),
		argst = std::forward_as_tuple(std::forward<Args>(args)...)
	]<typename... Kwargs>(Kwargs&&... kw) {
		return std::apply(
			std::get<0>(std::move(ft)),
			std::tuple_cat(std::move(argst), std::forward_as_tuple(std::forward<Kwargs>(kw)...))
		);
	};
}

int main() {
	auto f = [](int x, double y) { return x * y; }
	auto g = curry(f, 10);
	std::cout << g(2.2);
}
```



deduction guide in C++17 ==TODO==
tuples in C++11

==TODO== fill

```cpp
#include <type_traits>

template <typename T>
T&& Forward(std::remove_reference<T>& value) {  // we can't accept rvalue
  return static_cast<T&&>(value);
}

// forward<int&> -> int& && -> int& => int& forward(...)
// forward<int> -> int && -> int&& => int&& forward(...)

template <typename T>
T&& Forward(std::remove_reference<T>&& value) {
  return static_cast<T&&>(value);
}

template <typename T>
void f(T&& value) {
  Forward<T>(value);
}

int main() {
  f(10);  // T=int
  int x = 1;
  f(x);  // T=int&
}

```
