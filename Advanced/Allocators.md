- Читается в процессе изучения `Memory`
- `new` - это абстракция над выделением памяти в C и работать с ней не очень удобно. Например: хотим для типа `MyType` чтобы `std::vector` выделял память одним способом, а `std::set` другим. С помощью перегрузок оператора new этого не добиться
- Поэтому в C++ появилась более высокоуровневая абстракция: аллокаторы. Это способ переопределить выделение памяти до момента обращения к оператору `new`

# `std::allocator`
- [See more](https://en.cppreference.com/w/cpp/memory/allocator)

```cpp
template <typename T>
struct Allocator {
	T* allocate(size_t n) {
		return ::operator new(n * sizeof(T), static_cast<std::align_val_t>(alignof(T)));  // see alignment
	}
	
	void deallocate(T* ptr, size_t) {
		::operator delete(ptr);
	}
	
	template <typename... Args>
	void construct(T* ptr, Args&&... args) { // Args&&`
	    new(ptr) T(args...); // std::forward(args)
	}

	void destroy(T* ptr) noexcept {
		ptr->~T();
	}
}
```

# `allocator_traits`
- [See more](https://en.cppreference.com/w/cpp/memory/allocator_traits)
- Большинство методов у всех аллокаторов будут одинаковые (например `construct` и `destroy`) а еще внутри есть куча `using`, поэтому в C++ была добавлена специальная обертка `allocator_traits`
- `allocator_traits` -  это структура, которая шаблонным параметром принимает класс вашего аллокатора
- Почти все методы и внутренние юзинги работают по прицнипу "взять у аллокатора если есть, если нет сгенерировать автоматически"

### `Vector` with `allocator_traits`

```cpp
#include <memory>
#include <stdexcept>

template <typename T, typename Alloc = std::allocator<T>>
class Vector {
 public:
  Vector(size_t n, const T& value = T(), const Alloc& allocator = Alloc());

  T& operator[](size_t i) { return arr_[i]; }

  const T& operator[](size_t i) const { return arr_[i]; }

  T& at(size_t i) {
    if (i >= size_) {
      throw std::out_of_range("...");
    }

    return arr_[i];
  }

  const T& at(size_t i) const {
    if (i >= size_) {
      throw std::out_of_range("...");
    }

    return arr_[i];
  }

  size_t size() { return size_; }
  size_t capacity() { return capacity_; }

  void resize(size_t n, const T& value = T());

  void reserve(size_t n);

 private:
  using AllocTraits =
      std::allocator_traits<Alloc>;  // это не обязательно, но так будет удобней

  T* arr_;
  size_t size_;
  size_t capacity_;
  Alloc alloc_;  // Сохраняем к себе!
};

template <typename T, typename Alloc>
void Vector<T, Alloc>::reserve(size_t n) {
  if (n <= capacity_) {
    return;
  }
  // T* new_arr = new T[n]; // На неуд
  // T* new_arr = reinterpret_cast<T*>(new int8_t[n * sizeof(T)]); На удос
  // T* new_arr = alloc_.allocate(n); На хор
  T* new_arr = AllocTraits::allocate(alloc_, n);  // На отл

  size_t i = 0;
  try {
    for (; i < size_; ++i) {
      AllocTraits::construct(alloc_, new_arr + i,
                             arr_[i]);  // здесь std::move(arr_[i]) (а точнее move_if_noexcept)
    }
  } catch (...) {
    for (size_t j = 0; j < i; ++j) {
      AllocTraits::destroy(alloc_, new_arr + j);
    }
    AllocTraits::deallocate(alloc_, n);
    throw;
  }

  for (size_t i = 0; i < size_; ++i) {
    AllocTraits::destroy(alloc_, arr_ + i);
  }

  AllocTraits::deallocate(alloc_, arr_, capacity_);

  arr_ = new_arr;
  capacity_ = n;
}

```

# Rebinding allocators
- Example: in `List<T>` allocator is `Allocator<T>`, but we need to allocate `Note<T>`
- Можно реализовать в аллокаторе и обращаться к ``Alloc::rebind<Node<T>::other`
```cpp
class Allocator {
  template <typename U>
  struct rebind {using other = Allocator<U>; }
}
```
- Но это неудобно, да и реализация почти всегда будет одинаковой у всех аллокаторов, поэтому это вынесли в `allocator_traits`
- Realized in `std::allocator_traits`
```cpp
template <typename T, typename Alloc = std::allocator<T>>
class List {
 public:
  // ...
 private:
  using AllocTraits = std::allocator_traits<Alloc>;
  using NodeAlloc = typename AllocTraits::template rebind_alloc<Node>;
  using NodeAllocTraits = typename AllocTraits::template rebind_traits<Node>; // same as std::allocator_trais<NodeAlloc>

  NodeAlloc alloc_;
};
```

# Copying allocators
```cpp
PoolAlloc alloc1;
PoolAlloc alloc2 = alloc1;
```
- Не знаем, что требуется:
	- Что `alloc2` просто должен работать как `PoolAlloc` и скопировал в себя настройки (размер пула и тд) из `alloc1`
	- Или чтобы эти два аллокатора отвечали за один и тот же пул
- Для это есть несколько юзингов

# Устройство аллокатора
- [More about](https://en.cppreference.com/w/cpp/memory/allocator)

| Type                                     | Definition                                                                                                      | Desc                                                 |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `value_type`                             | `T`                                                                                                             |                                                      |
| `pointer`                                | `T*`                                                                                                            | - deprecated in C++17<br>- removed in C++20          |
| `const_pointer`                          | `const T*`                                                                                                      | - deprecated in C++17<br>- removed in C++20          |
| `reference`                              | `T&`                                                                                                            | - deprecated in C++17<br>- removed in C++20          |
| `const_reference`                        | `const T&`                                                                                                      | - deprecated in C++17<br>- removed in C++20          |
| `size_type`                              | `std::size_t`                                                                                                   |                                                      |
| `difference_type`                        | `std::ptrdiff_t`                                                                                                |                                                      |
| `propagate_on_container_move_assignment` | `std::true_type`                                                                                                | C++11                                                |
| `rebind`                                 | ```cpp<br>template< class U >  <br><br>struct rebind  <br>{  <br>    typedef allocator<U> other;  <br>};<br>``` | - deprecated in C++17<br>- removed in C++20          |
| `is_always_equal`                        | `std::true_type`                                                                                                | C++11<br>- deprecated in C++23<br>- removed in C++26 |

- `is_always_equal : true_type`
	- Всегда равен любому другому
	- about `true_type` [see](https://en.cppreference.com/w/cpp/types/integral_constant)
- `propagate_on_container_copy/move/swap_assignment : true_type`
	- Говорит, надо ли при копировании/перемещении/swap'а объекта перетягивать и аллокатор (pool allocator for example)
	- Но все равно аллокатор перетягиваются, если аллокаторы не равны:
```cpp
S& operator=(other) {
	if(p_o_c_c_a || (!is_always_equal && alloc != other.alloc)) {
		alloc = other.alloc;
	}
	...
} // this cringe + million of try-catch
```

# Устройство `std::allocator_traits`

### Member types (самое интересное)

| `allocator_type`                         | `Alloc`                                                                                 |
| ---------------------------------------- | --------------------------------------------------------------------------------------- |
| `value_type`                             | `Alloc::value_type`                                                                     |
| `pointer`                                | `Alloc::pointer` if present, otherwise `value_type`                                     |
| `propagate_on_container_copy_assignment` | `Alloc::propagate_on_container_copy_assignment` if present, otherwise `std::false_type` |
| `propagate_on_container_move_assignment` | `Alloc::propagate_on_container_move_assignment` if present, otherwise `std::false_type` |
| `propagate_on_container_swap`            | `Alloc::propagate_on_container_swap` if present, otherwise `std::false_type`            |
| `is_always_equal`                        | `Alloc::is_always_equal` if present, otherwise `std::is_empty<Alloc>::type`             |

### Member functions (самое интересное)

| Function                                       | Desc                                                            |
| ---------------------------------------------- | --------------------------------------------------------------- |
| `static allocate`                              | allocates uninitialized storage using the allocator             |
| `static deallocate`                            | deallocates storage using the allocator                         |
| `static construct`                             | constructs an object in the allocated storage                   |
| `static destroy`                               | destructs an object stored in the allocated storage             |
| `static select_on_container_copy_construction` | obtains the allocator to use after copying a standard container |

# Полиморфные аллокаторы

- Вообще раньше аллокаторы были stateless (без состояния => без полей)
- Было сложно работать с разными типами: `vector<int, Alloc1> =???= vector<int, Alloc2>`

```cpp
template <typename T>
struct polymorphic_allocator : std::scoped_allocator_adapter<...> {

	std::memory_resource* resource_;
};

struct memory_resource {
	virtual void* allocate(size_t s) = 0;
	virtual void deallocate(void* p) = 0;
	virtual bool is_equal(other) = 0;
};
```

- `memory_resource` без шаблонов => можно сделать `virtual allocate/deallocate/... methods` и делать наследников:
	- `struct std::null_resource : std::memory_resource { ... }`
		- Чистая заглушка
	- `struct std::new_delete_resource : std::memory_resource { ... }`
		- Честный `new` + `delete`
		- Вообще это не наследник, а отдельная функция (все равно оно stateless)
	- `get/set_default_resource()` - функции управления ресурсов по умолчанию (далее пригодится)
		- Изначально - `new_delete_resource`
	- `struct monotonic_buffer_resource : std::memory_resource { ... }`
		- Есть большой пулл, он вытаскивает память из него
		- Когда закончился буффер, обращается к `upstream_resource` - указателю на еще один `memory resource`. По умолчанию - `get_default_resource()`
	- `struct std::(un)syncronized_pool_resource : std::memory_resource { ... }`
		- Выделяет несколько буфферов (пуллов) (вроде 4)
		- Первый пулл - для 1 аллокации 1 байта
		- Второй - 2-х
		- Третий - 4-х
		- Четвертый - для аллокации памяти размерности более 4-х байт
		- `syncronized` расчитан для многопотока, а `unsyncronized` быстрее (не тратит много времени на синхронизацию), в остальном они одинаковы

### Создаем свой resource
```cpp
struct std::memory_resource {
	void* allocate(size_t) {
		return do_allocate(s);
	}

	void* do_allocate(size_t) = 0;
};

struct my_res : public std::memory_resource {
	void* do_allocate(size_t) { // Uses NVI
		// ...
	}
};
```

### propagate_on_container_X_assignment
- Пропагейты - `on_container_copy/move/swap`
- У полиморфика все выставлены в `false`

```cpp
namespace std::pmr {
	template <class T>  
    using vector = std::vector<T, std::pmr::polymorphic_allocator>; // c++17
}
```

```cpp
pmr::vector<pmr::vector<int>> a(3);

[v1] -> A1
[v2] -> A2
[v3] -> A3
[-] -> -

insert(begin(), pmr::vector());

[v4] -> A1
[v1] -> A2
[v2] -> A3
[v3] -> A3

insert(begin(), pmr::vector());
// realloc (first of all, will insert in new data ptr new item (for safety exceptions))
[v5] -> A5
[v4] -> A4
[v1] -> A2
[v2] -> A3
[v3] -> A3

// cringe
```

```cpp
pmr::vector<int> v = f();  // assign f() allocator quickly

pmr::vector<int> u;  // creates own allocator
u = f();  // ?! - different behaviour
```
