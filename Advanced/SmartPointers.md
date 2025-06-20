Умные указатели - указатели, которые следят за освобождением ресурса. Построены на идиоме RAII.
- Умные указатели - это основной инструмент RAII для управления динамически выделяемой памятью в C++.

# RAII
- _Resource Acquisition Is Initialization_, or RAII, is a C++ programming technique which binds the life cycle of a resource that must be acquired before use (allocated heap memory, thread of execution, open socket, open file, locked mutex, disk space, database connection—anything that exists in limited supply) to the lifetime of an object.
### RAII solves problems
1. **Утечки памяти**: Без RAII разработчику приходится вручную отслеживать и освобождать выделенную память. Забытая операция освобождения памяти может привести к утечкам. RAII гарантирует, что память будет автоматически освобождена при уничтожении объекта.

2. **Неопределенное поведение**: Если ресурсы не управляются должным образом, это может привести к неопределенному поведению программы. RAII гарантирует, что ресурсы всегда находятся в определенном состоянии.

3. **Исключения и безопасность**: RAII позволяет обрабатывать исключения более элегантно и безопасно. Ресурсы будут автоматически освобождены даже в случае возникновения исключительных ситуаций.

```cpp
struct FileHandler {
 public:
  FileHandler() : data(new int[100]) {
    // something that can throw exception
  }

  ~FileHandler() { delete[] data; }

  int* data;
};
```

- [Source](https://habr.com/ru/companies/otus/articles/778942/)

# `unique_ptr`
`unique_ptr` - простейший умный указатель. Он запрещает себя копировать и удаляет ресурс в деструкторе.
- Простота именно в запрете на копирование (реализовать `shared_ptr`, где несколько объектов владеют ресурсом куда сложней)

Методы `unique_ptr`:
- `->` и `*`: `unique_ptr` работает как указатель
- `get`: отдает "сырой" указатель который лежит под юником
- `release`: отдает "сырой" указатель и делает юник пустым
- `swap`: меняет ресурс под юником
- `operator bool`: проверка на `nullptr`

```cpp
template <typename T>
class UniquePtr {
 public:
  UniquePtr(T* ptr): ptr_(ptr) {}

  ~UniquePtr() {
    delete ptr_;
  }

  UniquePtr(const UniquePtr&) = delete;  // запрещаем копироваться
  UniquePtr& operator=(const UniquePtr&) = delete;

  UniquePtr(UniquePtr&& other): ptr_(other.ptr_) {other.ptr_ = nullptr; }
  UniquePtr& operator=(UniquePtr&&) {
    if (this == &other) {
      return *this;
    }
    delete ptr_;
    ptr_ = nullptr;
    std::swap(ptr_, other.ptr_);
    return *this;
  }

 private:
  T* ptr_;
};
```
- Остальные методы очев

### Some moments
1. `std::unique_ptr<int> p(new int[5]);` не будет нормально работать, так как в деструкторе будет вызван `delete p`. Нужно делать так: `std::unique_ptr<int[]>p(new int[5]);`: начиная с C++17 для специализации с массивами есть квадратные скобки
2. Даже константный `unique` возвращает просто указатель и ссылку (ввиду определения константности для классов)

### `deleter`
- True `unique_ptr` signature is:
```cpp
template <typename T, typename Deleter = std::default_delete<T>>
class UniquePtr;
```
- Потому что иногда может возникнуть ситуация, что нужен кастомный `deleter` для вашего умного указателя
- Вызывается с помощью `void operator(T* ptr);`

# `shared_ptr`
- `shared_ptr` позволяет нескольким указателям владеть одним ресурсом. При этом ресурс удаляется только после уничтожения всех "владельцев"
- Методы у него примерно такие же как у `unique_ptr`
- Также `shared_ptr` должен уметь работать с кастомными `Deleter` и `Allocator`. Они бросаются в конструктор.

- Идея такова: в самом `shared_ptr` мы будем хранить указатель на `BaseControlBlock`
```cpp
template <typename T>
struct BaseControlBlock {
  T object;
  size_t shared_count;
  size_t weak_count;

  BaseControlBlock(const T&, size_t, size_t);
  virtual ~BaseControlBlock() = default;
}

template <typename Deleter>
struct ControlBlock: BaseControlBlock {
  Deleter deleter;
  ControlBlock(const T&, size_t, size_t, const Deleter& deleter);
}
```

# `std::make_shared`, `std::make_unique`
- Раз работаем с "высокоуровневыми" указателями, хочется полностью обезопаситься и исключить из обихода `new [bla-bla-bla]`
```cpp
int main() {
  int* p = new int(5);
  std::shared_ptr<int> sp1(p);
  std::shared_ptr<int> sp2(p);  // UB, double free
}
```

### `std::make_unique`
```cpp
template <typename T, typename... Args>
uniqe_ptr<T> make_unique(Args&&... args) {
  return unique_ptr<T>(new T(std::forward<Args>(args)...));
}

int main() {
  auto p = std::make_unique<int> p(5);
}
```

### `std::make_shared`
```cpp
template <typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args) {
  return shared_ptr<T>(details::MakeSharedTag{}, std::forward<Args>(args)...);
}
```

# `weak_ptr`
### Problem
```cpp
struct S {
  std::shared_ptr<S> ptr;
}
int main() {
  auto s1 = std::make_shared<S>();
  auto s2 = std::make_shared<S>();
  s1.ptr = s2;
  s2.ptr = s1;
}
```
- В данном случае получится утечка памяти
- `Shared` никогда не удалится
- Данная проблема может показаться надуманной, однако возникает очень много ситуаций в реальном мире, где эта проблема актуальна (например хранение родителя в дереве, хранение указателя на предыдущий элемент в списке).
- Из-за этой проблемы сборка мусора это не очень тривиальная задача. Одно из решений этой проблемы это "слабые ссылки". В случае C++ это `weak_ptr`

### Solution
- `weak_ptr` хоть и назван как указатель, однако он не ведет себя как таковой. Его нельзя разименовывать!
- Он умеет делать только две вещи:
	1. Отвечать на вопрос "Сколько еще шаредов указывает на объект"
	2. По запросу создавать новый шаред на объект (опять-таки, если объект еще жив, т.е. есть шаред, на него ссылающийся)
- Т.е. `weak_ptr` не владеет объектом и не учавствует в подсчете ссылок. Такая конструкция решает наши проблемы: просто используем `weak_ptr` для указания на родителей

```cpp
template <typename T>
class weak_ptr {
 public:
  weak_ptr(const shared_ptr<T>& ptr): helper_(ptr.helper_) {}

  bool expired() const { return helper_->count == 0; }

  shared_ptr<T> lock() const { return expired() ? nullptr : shared_ptr<T>(helper_); }

 private:
  ControlBlock<T>* helper_ = nullptr;
}
```

- Также в `ControlBlock` нужен еще один счетчик: `weak_count`.
- Теперь в деструкторе `shared_ptr` нужно удалять объект при обнулении `shared_count` и удалять весь блок при обнулении `shared_count + weak_count`. В `weak_ptr` в методе `expired` мы можем смотреть на `shared_count`, а в деструкторе удалять блок если `shared_count + weak_count` становится нулем

# Implementation
```cpp
#include <iostream>
#include <utility>

namespace details {

using counter_type_t = std::size_t;

template <typename T>
struct BaseControlBlock {
  BaseControlBlock() = default;
  BaseControlBlock(T* ptr) : ptr(ptr) {}
  virtual ~BaseControlBlock() { delete ptr; }  // But it's bad

  counter_type_t DecreaseStrongCount() { return --strong_count; }
  counter_type_t IncreaseStrongCount() { return ++strong_count; }

  T* ptr{nullptr};  // TE_REMOVE WITHOUT
  counter_type_t strong_count{1};
  counter_type_t weak_count{0};  // to be continued...
};

template <typename T>
struct ControlBlock : BaseControlBlock<T> {
  template <typename... Args>
  ControlBlock(Args&&... args) : value(std::forward(args)...) {
    this->ptr = &value;
  }

  ~ControlBlock() override {}

  T value;
  /*
  // can: we want to call T value destructor
  union {
    T value;
    // std::byte trash[sizeof(T)];  // optional (can left only T value; - union
    // won't call T destructor)
  }
  */
};

struct MakeSharedTag {};

}  // namespace details

template <typename T>
class SharedPtr {
 public:
  using counter_type_t = details::counter_type_t;

  SharedPtr() = default;
  SharedPtr(T* ptr) { contol_block_ = new details::BaseControlBlock<T>(ptr); }

  SharedPtr(const SharedPtr& other) noexcept {
    if (other.contol_block_ == nullptr) {
      return;
    }
    contol_block_ = other.contol_block_;
    contol_block_->IncreaseStrongCount();
  }

  SharedPtr(SharedPtr&& other) noexcept {
    other.contol_block_ = std::exchange(contol_block_, other.contol_block_);
  }

  SharedPtr& operator=(const SharedPtr& other) noexcept {
    if (contol_block_ ==
        other.contol_block_) {  // may not write this == std::adressof(other)
      return *this;
    }
    auto count = DecreaseStrongCount();
    if (count == 0) {
      delete contol_block_;
    }
    contol_block_ = other.contol_block_;
    IncreaseStrongCount();
    return *this;
  }

  SharedPtr& operator=(SharedPtr&& other) noexcept {
    if (contol_block_ == other.contol_block_) {
      return *this;
    }

    auto count = DecreaseStrongCount();
    if (count == 0) {
      delete contol_block_;
    }

    contol_block_ = std::exchange(other.contol_block_, nullptr);
  }

  ~SharedPtr() noexcept {
    counter_type_t counter = DecreaseStrongCount();
    if (counter == 0) {
      delete contol_block_;
    }
  }

  T& operator*() { return *contol_block_->ptr; }
  const T& operator*() const { return *contol_block_->ptr; }

  T* operator->() noexcept { return contol_block_->ptr; }
  const T* operator->() const noexcept { return contol_block_->ptr; }

 private:
  template <typename... Args>
  SharedPtr(details::MakeSharedTag, Args&&... args) {
    contol_block_ = new details::ControlBlock<T>(std::forward<Args>(args)...);
  }

  counter_type_t DecreaseStrongCount() {
    if (contol_block_ == nullptr) {
      return 0;
    }
    return contol_block_->DecreaseStrongCount();
  }

  counter_type_t IncreaseStrongCount() {
    if (contol_block_ == nullptr) {
      return 0;
    }
    return contol_block_->IncreaseStrongCount();
  }

  // TE_ADD T* ptr; (aliasing contructors)
  // We can add T* ptr; (for operator* one heap-casting)
  details::BaseControlBlock<T>* contol_block_{nullptr};

  template <typename... Args>
  friend SharedPtr<T> MakeShared(Args&&... args);
};

template <typename T, typename... Args>
SharedPtr<T> MakeShared(Args&&... args) {
  return SharedPtr<T>(details::MakeSharedTag{}, std::forward<Args>(args)...);
}

struct S {
  int x;
  int y;
};

int main() {
  // std::shared_ptr<S> sp = std::make_shared<S>(S{10, 20});
  // std::shared_ptr<int> si = std::shared_ptr<int>(&(sp->x));  // UB

  SharedPtr<int> sp1(new int(10));
  {
    auto sp2(sp1);
    auto sp3(sp2);

    std::cout << *sp2 << '\n';
  }
  std::cout << *sp1 << '\n';

  return 0;
}
```

# Comparison table and `boost` attachments

| `X_ptr`                                                                                                              | move supported | copy supported |
| -------------------------------------------------------------------------------------------------------------------- | -------------- | -------------- |
| `std::unique_ptr`                                                                                                    | +              | -              |
| `std::shared_ptr`                                                                                                    | +              | +              |
| `boost::scoped_ptr`<br>- equals a `const std::unique_ptr`                                                            | -              | -              |
| `boost::intrusive_ptr`<br>- only for `T` contains own `counter`<br>- interface: `DecreaseCount()`, `IncreaseCount()` | +              | +              |

# `enable_shared_from_this`

### Problem
- Хотим изнутри класса возвращать указатель на самого себя
```cpp
struct Test {
	std::shared_ptr<Test> GetSelfPtr() {
		return std::shared_ptr<Test>(this);
	}
};

int main() {
	auto t = std::make_shared<Test>();
	t->GetSelfPtr();  // Double free
}
```

- Даже если в полях хранить `std::shared_ptr`, то объект вообще никогда не удалится (будет закольцовано)
- Можем хранить `std::weak_ptr<Test> ptr_;`. Тогда надо заполнять этот `weak_ptr` при каждом создании. Плюсом, каждый раз прописывать логику для каждого класса - не есть хорошо

### Solution
- Здесь мы познакомимся с приемом CRTP (Curiously recursive template pattern)
	- ==TODO== checkout, name shared from this

```cpp
template <typename T>
struct Base {
	void print() {
		static_cast<T&>(*this).print();
	}
};

struct Derived : public Base<Derived> {
	void print() {
		std::cout << "D";
	}
};

int main() {
	Derived d;
	Base<Derived>& b = d;
	b.print();  // D
}
```

- Стандарт позволяет наследоваться от шаблонного класса, в качестве шаблонного параметра которого указывается сам класс
- То есть суть CRTP такова: наследуемся от шаблона с шаблонными параметром себя

- Решением нашей проблемы будет наследование от `std::enable_shared_from_this<T>`
```cpp
struct S: public std::enable_shared_from_this<S> {
  shared_ptr<S> GetPtr() const {
    return shared_from_this(); // метод появляется из enable_shared_from_this
  }
}
```

- `std::enable_shared_from_this` может быть интерпретирован следующим образом
```cpp
template <typename T>
class enable_shared_from_this {
 protected:
  shared_ptr<T> shared_from_this() const {
    return ptr_.lock();
  }
 private:
  weak_ptr<T> ptr_ = nullptr;
}
```

- Тогда при конструировании `std::shared_ptr` необходимо фиксироваться для создаваемого объекта
```cpp
shared_ptr(T* ptr) {
    if constexpr (std::is_base_of<enable_shared_from_this<T>, T>) {
        ptr->ptr_ = std::weak_ptr(ptr);
    }
}
```
- На самом деле делать это надо и в конструкторе от `ControlBlock` тоже. Ну и конструктор `weak_ptr` нужен соответствующий

# Касты умных указателей
- Рассмотрим код
```cpp
struct Base {};
struct Derived: public Base {};

int main() {
    auto ptr = std::make_shared<Derived>();
    std::shared_ptr<Base> b_ptr = ptr;
}
```
- Он это умеет

### All smart pointer casts
- [Docs](https://en.cppreference.com/w/cpp/memory/shared_ptr/pointer_cast)
- ==TODO== examples in docs
```cpp
std::static_pointer_cast
std::dynamic_pointer_cast
std::const_pointer_cast
std::reinterpret_pointer_cast
```

- Possible implementations
```cpp
template<class T, class U>
std::shared_ptr<T> static_pointer_cast(const std::shared_ptr<U>& r) noexcept {
    auto p = static_cast<typename std::shared_ptr<T>::element_type*>(r.get());
    return std::shared_ptr<T>{r, p};  // aliasing constructor
}


template<class T, class U>
std::shared_ptr<T> dynamic_pointer_cast(const std::shared_ptr<U>& r) noexcept {
    if (auto p = dynamic_cast<typename std::shared_ptr<T>::element_type*>(r.get()))
        return std::shared_ptr<T>{r, p};
    else
        return std::shared_ptr<T>{};
}


template<class T, class U>
std::shared_ptr<T> const_pointer_cast(const std::shared_ptr<U>& r) noexcept {
    auto p = const_cast<typename std::shared_ptr<T>::element_type*>(r.get());
    return std::shared_ptr<T>{r, p};
}


template<class T, class U>
std::shared_ptr<T> reinterpret_pointer_cast(const std::shared_ptr<U>& r) noexcept {
    auto p = reinterpret_cast<typename std::shared_ptr<T>::element_type*>(r.get());
    return std::shared_ptr<T>{r, p};
}
```
