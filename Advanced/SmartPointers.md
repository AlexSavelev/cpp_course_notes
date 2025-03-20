Умные указатели - указатели которые следят за освобождением ресурса. Построены на идиоме RAII.
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

  UniquePtr(const UniquePtr&) = delete; // запрещаем копироваться
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
1. `std::unique_ptr<int> p(new int[5]);` не будет нормально работать, так как в деструкторе будет вызван `delete p`. Нужно делать так: `std::uniqe_ptr<int[]>p(new int[5]);`
2. А вообще начиная с C++17 для специализации с массивами есть квадратные скобки
3. Даже константный `unique` возвращает просто указатель и ссылку (ввиду определения константности для классов)

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

```cpp
template <typename T>
class shared_ptr {
 public:
  explicit shared_ptr(T* ptr) : ptr(ptr), count(new int(1)) {}

  ~shared_ptr() {
    if (count == nullptr) {
      return;
    }
    --*count;
    if (*count == 0) {
      delete ptr;
      delete count;
    }
  }

 private:
  T* ptr = nullptr;
  size_t* count = nullptr;
};
```
- Остальное очев

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
  return uniqe_ptr<T>(new T(std::forward<Args>(args)...));
}

int main() {
  auto p = std::make_unique<int> p(5);
}
```

### `std::make_shared`
```cpp
template <typename T>
class shared_ptr {
 public:
  explicit shared_ptr(T* ptr);  // How to do?! To be continued...

  ~shared_ptr() {
    if (control_block_ == nullptr) {
      return;
    }

    --control_block_->count;

    if (control_block_->count == 0) {
      delete control_block_;
    }
  }

 private:
  template <typename U>
  struct ControlBlock {
    U object;
    size_t count;
  }

  template <typename U, typename... Args>
  friend shared_ptr<U> make_shared(Args&&... args);

  shared_ptr(ControlBlock<T>* ptr): control_block_(control_block_) {}

  ControlBlock<T>* control_block_;
};

template <typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args) {
  auto p = new ControlBlock(1, std::forward<Args>(args)...);
  return shared_ptr<T>(p);
}
```

#### Решение для конструктора от `ptr`
```cpp
template <typename T>
struct BaseControlBlock {
  T* ptr;
  size_t* count;
};
template <typename T>
struct MakeSharedControlBlock: BaseControlBlock<T> {
  T obj;
  MakeSharedControlBlock(): ptr(&obj) {}
};
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
	2. По запросу создавать новый шаред на объект
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
- Теперь в деструкторе `shared_ptr` нужно удалять объект при обнулении `shared_count` и удалять весь блок при обнулении `weak_count`. В `weak_ptr` в методе `expired` мы можем смотреть на `shared_count`, а в деструкторе удалять блок если `weak_count` становится нулем
