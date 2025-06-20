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

# Union
- Этот функционал мы унаследовали из C. Можно объявить такой тип данных
```cpp
union U {
  int x;
  double y;
  char c;
};
```

- Когда мы просто объявляем юнион активным членом становится первый член
```cpp
int main() {
  U u;  // в данный момент лежит int
  std::cout << u.x;  // 0
  u.d = 3.14;  // положили дабл
  std::cout << u.d;  // вывело 3.14
  std::cout << u.x;  // UB! (на самом деле будет реинтерпрет каст)
}
```

- Проблемы появляются когда хотим положить в union поле с нетривиальными конструкторами/деструкторами, например `std::string`
```cpp
union U {
  int x;
  double y;
  std::string s;
  U(const char* s): s(s) {}
  ~U() {}
};
```

- Возникает следующая проблема: после смены активного члена юнион не уничтожает предыдущий активный член. То есть в следующем коде есть утечка памяти:
```cpp
union U {
  int x;
  double d;
  std::string s;

  U(const char* s): s(s) {}
  ~U() {}
};

int main() {
  U u = "abc";
  std::cout << u.s;
  u.d = 3.14;
}
```

- Чтобы это поправить придется вызвать явно деструктор строки:
```cpp
  U u = "abc";
  u.s.~basic_string();
  u.d = 3.14;
```

- Следующая проблема
```cpp
union U {
  int x;
  double y;
  std::string s;

  U(double d): y(d) {}
  U(const char* s): s(s) {}
  ~U() {}
};

int main() {
  U u = 3.14;
  u.s = "abc";
}
```

- Будет сегфолт потому что под `u.s` лежит какой-то мусор, а мы у него вызываем `оператор=`. Придется воспользоваться `placement new`

```cpp
union A {
   A () { new (&s1) std::string ("hello world"); }
  ~A () { s1.~basic_string<char> (); }

  int         n1;
  std::string s1;
};
```

```cpp
new (&u.s) std::string("Test");
std::cout << u.s;
```

- Стоит так же упомянуть про анонимные юнионы:
```cpp
int main() {
  union {
    int a;
    double b;
  };

  a = 10;
  b = 20;
}
```

# `std::variant`
```cpp
int main() {
  std::variant<int, double, std::string> v;
  v = "abc";

  std::cout << std::get<std::string>(v);
  std::cout << std::get<int>(v); // std::bad_variant_access

  v = 5; // int положился и строка корректно уничтожился
}
```

### Implementation
```cpp
template <typename... Types>
union VariadicUnion {
  // тут бы надо определить что такое get от пустого юниона

  template <typename T>
  void put(const T&) {
    static_assert(false, "fail"); // костыль
  }
};

template <typename Head, typename... Tail>
union VariadicUnion<Head, Tail...> {
  Head head;
  VariadicUnion<Tail...> tail;

  VariadicUnion() = default;
  ~VariadicUnion() = default;

  template <size_t N>
  auto& get() {
    if constexpr (N == 0) {
      return head;
    } else {
      return tail.template get<N>();
    }
  }

  template <typename T>
  void put(const T& value) {
    if constexpr (std::is_same_v<T, Head>) {
      new(&head) T(value);
    } else {
      tail.template put(value);
    }
  }
};

template <typename T, typename... Types>
struct VariantAlternatives {  // новая структура
  using Derived = Variant<Types...>;

  static constexpr index = index_by_type<T, Types...>;

  VariantAlternatives(const T& value) {
    auto variant = static_cast<Derived*>(this);
    variant->storage.template put<T>(value);
    ptr->current_index = index;
  }

  VariantAlternatives() = default;
  ~VariantAlternatives() = default;
};

template <typename... Types>
struct Variant
  : public VariantAlternatives<Types, Types...>... {  // наследование!
  VariadicUnion<Types...> storage;

  ~Variant() {
    (VariantAlternatives<Types, Types...>::destroy(), ...);
  }

  size_t current_index = 0;
  using VariantAlternatives<Types, Types...>::VariantAlternatives...;  // работает с g++-11
};

template <size_t N, typename... Types>
auto& get(Variant<Types...>& v) {
  if (v.current_index == N) {
      return storage.template get<N>();
  }
  throw std::bad_variant_access();
}

// Чтобы сделать get по типу придется написать метафункцию index_by_type которая из пакета аргументов достает i-й тип.

template <size_t N, typename T, typename... Types>
struct index_by_type_impl {
  static constexpr size_t value = 0;
};

template <size_t N, typename T, typename Head, typename... Tail>
struct index_by_type_impl<N, T, Head, Tail...> {
  static constexpr size_t value = std::is_same_v<T, Head> ? N : index_by_type_impl<N+1, T, Tail...>::value;
};

template <typename T, typename... Types>
static constexpr size_t index_by_type = index_by_type_impl<0, T, Types...>::value;

// Тогда get по типу реализуется очевидно.

// В целом это идейно и есть реализация варианта. Правда у нас тут есть UB. А именно в строчке variant->storage.template put(value);
// В этот момент мы в конструкторе родителя, то есть еще не заинитен наследник. То есть поля storage еще не существует))). Выход следующий: сначала отнаследуемся от VariadicStorage а потом от всех наших родителей.
```
