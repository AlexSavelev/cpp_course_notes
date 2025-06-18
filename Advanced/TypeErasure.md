- Type erasure - техника затирания типов
- Примеры: `std::any`, `std::function`, `void*` (and `std::byte`), `std::span`

# `std::any`
- Структура данных, которая позволяет хранить в себе что угодно

```cpp
#include <any>
#include <vector>

int main() {
    std::any a = 5;
    std::vector<int> v = {1, 2, 3, 4 ,5};
    a = std::move(v);
    a = 'a';

    std::cout << std::any_cast<char>(a);
    // Если мы "не угадаем" тип, то будет exception std::bad_any_cast
}
```

### Possible implementation
```cpp
struct Any {
  struct Base {
    virtual ~Base() = default;

    virtual Base* GetCopy() const = 0;
  };

  template <typename T>
  struct Derived : public Base {
    T object;
    Derived(const T& object): object(object) {}

    Base* GetCopy() const override {
      return new Derived<T>(object);
    }
  };

  template <typename U>
  Any(const U& object) : ptr(new Derived<U>(object)) {}

  Any(const Any& other) : ptr(ptr->GetCopy()) {}

  template <typename U>
  Any& operator=(const U& object) {
	// Must fix copy-assignment operator for any (compare this...)
    delete ptr;
    ptr = new Derived<U>(object);
  }

  ~Any() {
    delete ptr;
  }

  Base* ptr = nullptr;
};

template <typename T>
T& AnyCast(Any& a) {
	auto d_ptr = dynamic_cast<Derived<T>*>(a.ptr);
	if (d_ptr == nullptr) {
		throw std::bad_any_cast();
	}
	return d_ptr->obj;
}
```

# `std::function`
### Usage
```cpp
#include <functional>
#include <iostream>

int foo(int x) { return x * x; }

int bar(int x) { return x + x; }

struct Callable {
  int operator()(int x) { return x * y; }

  int method(double x) { return x * x; }

  int y = 20;
};

int main() {
  std::function<int(int)> f = foo;
  std::cout << f(10) << '\n';
  f = bar;
  std::cout << f(10) << '\n';
  Callable c;
  f = c;
  std::cout << f(10) << '\n';

  auto yptr = &Callable::y;
  std::cout << std::invoke(yptr, c) << '\n';
  c.*yptr;
  std::function<int(Callable&)> f1 = &Callable::y;
  std::cout << f1(c) << '\n';

  auto method_ptr = &Callable::method;

  std::cout << std::invoke(method_ptr, c, 2.5) << '\n';
  (c.*method_ptr)(2.2);
  std::function<int(Callable&, double)> f3 = &Callable::method;
  std::function f4 = [&](int y) { return y * y * y; };
}
```

- То есть `std::invoke` вызывает не только функции, но и вычисляет поле класса по данному проектору

```cpp
#include <functional>
#include <iostream>

struct S {
  void foo(int y) { std::cout << x * y; }
  int x;
};

int main() {
  auto SFoo = &S::foo;
  S s{10};

  auto SP = &S::x;
  std::invoke(SFoo, s, 20);
  std::cout << std::invoke(SP, s) << '\n';
}
```

### Possible implementation
```cpp
#include <functional>
#include <iostream>
#include <type_traits>

template <typename T>
struct FunctionHolder {
  FunctionHolder(const T& value) : value(value) {}
  FunctionHolder(T&& value) : value(std::move(value)) {}

  T value;
};

template <typename R, typename F, typename... Args>
R call_function(void* callable, Args... args) {
  return std::invoke(static_cast<FunctionHolder<F>*>(callable)->value, args...);
}

template <typename F>
void delete_function(void* callable) {
  delete static_cast<FunctionHolder<F>*>(callable);
}

template <typename F>
void copy_function(void*& callable, void* other_callable) {
  // using SBO => no void* return (write-in-args)
  callable =
      new FunctionHolder<F>(*static_cast<FunctionHolder<F>*>(other_callable));
}

template <typename F>
class Function;

template <typename R, typename... Args>
class Function<R(Args...)> {
  using call_function_wrapper_t = R (*)(void*, Args...);
  using delete_function_wrapper_t = void (*)(void*);
  using copy_function_wrapper_t = void (*)(void*&, void*);

 public:
  template <typename F>
  // requires std::is_invocable_r_v<R, F, Args...>
  Function(F&& func) {
    callable_ = static_cast<void*>(
        new FunctionHolder<std::decay_t<F>>(std::forward<F>(func)));
    call_function_wrapper = &call_function<R, std::decay_t<F>, Args...>;
    delete_function_wrapper = &delete_function<std::decay<F>>;
    copy_function_wrapper = &copy_function<std::decay_t<F>>;
  }

  Function(const Function& other) {
    call_function_wrapper = other.call_function_wrapper;
    delete_function_wrapper = other.delete_function_wrapper;
    copy_function_wrapper = other.copy_function_wrapper;
    copy_function_wrapper(callable_, other.callable_);
  }

  Function& operator=(const Function& other) {
    if (this == std::addressof(other)) {
      return *this;
    }
    if (call_function_wrapper == other.call_function_wrapper) {
      // we can use trivial copy
      // must to do
    }
    delete_function_wrapper(callable_);
    other.copy_function_wrapper(callable_, other.callable_);
    call_function_wrapper = other.call_function_wrapper;
    delete_function_wrapper = other.delete_function_wrapper;
    copy_function_wrapper = other.copy_function_wrapper;
  }

  ~Function() {
    if (delete_function_wrapper) {
      delete_function_wrapper(callable_);
    }
  }

  R operator()(Args... args) {
    if (!call_function_wrapper) {
      throw "Bad invoke";
    }
    return call_function_wrapper(callable_, args...);
  }

 private:
  void* callable_ = nullptr;
  call_function_wrapper_t call_function_wrapper = nullptr;
  delete_function_wrapper_t delete_function_wrapper = nullptr;
  copy_function_wrapper_t copy_function_wrapper = nullptr;
};

int main() {
  Function<int(int)> f = [](int x) { return x * x; };
  std::cout << f(10) << '\n';

  return 1;
}
```
