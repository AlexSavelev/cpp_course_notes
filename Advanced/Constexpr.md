- Это про compile-time вычисления
# Compile-time вычисления на шаблонах
```cpp
template <int N, int Mod>
struct Factorial {
	const static int value = N * Factorial<N - 1, Mod>::value % Mod;
};

template <int Mod>
struct Factorial<1, Mod> {
	const static int value = 1;
};

static_assert(Factorial<1, 1>::value == 1);

int main() {
	return 0;
}
```

Вычисления на шаблонах - тьюринг полны. Но есть некоторые недостатки:
- Можем использовать ограниченний функционал: рекурсия, ифы
- Не можем нормально эмулировать циклы - ограничены глубиной рекурсии
- Код не интуитивный

# Constexpr вычисления
```cpp
constexpr int factorial(int n, int mod) {
    return n == 0 ? 1 : factorial(n - 1, mod) * n % mod;
}

static_assert(factorial(1, 1) == 1);
```

- Добавили к функции модификатор `constexpr`
	- Для функций - лишь указание, что ф-ция мб вычислена в compile-time
	- Для переменный - объявляем compile-time
- В C++11 это уже работает. Но можно писать только `return` что-то. Строчка может быть сколь угодно длинная, но должна быть всего одна
- В C++11 это работало как синтаксический сахар - все превращалось в шаблоны

```cpp
// В C++11 constexpr функция могла быть только одна строчка - с return
constexpr int foo1() {
	return 1;
}
```

Начиная с C++14 механизм `constexpr`-функций сильно расширили. Теперь факториал можно считать вот так:
```cpp
// В C++14 появился процесс "constant evaluation" - исполняется интерпретатором
constexpr int factorial(int n, int mod) {
    int result = 1;
    for (int i = 1; i <= n; ++i) {
        result *= i;
        result %= mod;
    }
    return result;
}
```

- Грубо говоря: завезли интерпретатор внутрь компилятора
- Этот процесс называется constant evaluation - отдельный этап компиляции, при котором запускается интерпретатор и отрабатывает на `constexpr`-функциях
	- Прямо как asan (address sanitizer)
	- В compile-time мы видим весь flow наших функций. В runtime - нет.

- Понятно, что нельзя использовать RT-объекты: `std::cout`
```cpp
constexpr int foo2() {
	if (x > 2) {
		return x * x;
	}
	std::cout << x;
	return 1;
}
```
- При этом `foo2(10)` - OK - вычисляется интерпретатором сверху-вниз
- `foo2(0)` - CE

- С C++20 можно писать внутри конструкции с `try` и `asm`
- С C++23 *вроде* можно писать RT-инструкции - компилятор будет игнорить

# Constexpr переменные
- Все константы можно разделить на 2 категории:
	1. Константы времени компиляции
	2. Константы времени исполнения
```cpp
#include <iostream>

int main() {
    const int a = 5;
    std::array<int, a> arr;  // OK

	int temp;
	std::cin >> temp;

	const int b = temp;
	std::array<int, b> arr2;  // CE
}
```

На этапе компиляции известны следующие вещи:
- Литералы (`1`, `1.0`, `1ull`, `'c'`, `"char"`)
- Параметры шаблонов
- Результат `sizeof`
- `constexpr`-переменные

Про последнее стоит отдельно поговорить
```cpp
int main() {
    constexpr int x = 2 * 2;
    std::array<int, x> arr; // OK
}
```
- На `constexpr` переменные накладывается следующее ограничение: `constexpr` переменная должна иметь литеральный тип
- Литеральный тип - это все базовые типы + некоторые другие, про которые поговорим позже

- _**Note**_ С C++20 даже `vector` является литеральным типом, но про это попозже

### `constexpr` - это не совсем `const`-qualifier
```cpp
constexpr int arr[] = {1, 2, 3};
constexpr int* x = &arr[3];
```

Непонятно: к чему в данном случае относится `constexpr`:
1. `constexpr int* x` -> `const int* x`
2. `constexpr int* x` -> `int* const`

```cpp
constexpr int arr[] = {1, 2, 3};
constexpr const int* x = &arr[3];  // добавили const
```

1. `constexpr int* x` -> `const int* x`
2. `constexpr int* x` -> `int* const` - правильный ответ

Вывод: `constexpr` не является частью типа, это аннотация (спецификатор)

- То есть `constexpr` не меняет тип объекта. Например, `constexpr int x = 10;` все еще является переменной типа `int`, но с дополнительным свойством возможности компиляции во время компиляции

# Constexpr функции
- Есть [перечисление](https://en.cppreference.com/w/cpp/language/constexpr) того, что можно делать в `constexpr` функциях
- `constexpr`-функции можно вызывать в runtime. Тогда они просто работают как обычные функции.
- Если функция используется в compile-time контексте, то ее вызов будет в compile-time. В любых других случаях гарантии не дается.
- _**Note**_ `constexpr` методы тоже можно (и во многих случаях даже нужно) объявлять. Про это будет позже

### `throw` в `constexpr`-функциях
- const evaluation выполняется интерпретатором => известен весь flow функции => почему бы не вызывать `throw()` ?
- `catch` запрещен
- Но все равно хороший способ ловить ошибки
```cpp
constexpr size_t Devide(size_t x, size_t y) {
    if (y == 0) {
        throw "y == 0";
    }
    return x / y;
}
```
- `error: expression ‘<throw-expression>’ is not a constant expression`

```cpp
#include <iostream>

struct DivisionByZero {};

constexpr int foo(int x, int y) {
	if (y == 0) {
		throw DivisionByZero{};
	}
	return x / y;
}

static_assert(foo(3, 3) == 1);
static_assert(foo(3, 0) == 1);  // CE: DivisionByZero is not a constant expression
```

```cpp
#include <iostream>

struct CheckIsNotZero {
  consteval CheckIsNotZero(int y) : y(y) {
    Check();
  }

  consteval void Check() {
    if (y == 0) {
      throw DevisionByZero{};
    }
  }

  int y = y;
};

int bar(int x, CheckIsNotZero checked) {
  std::cout << x << '\n';
  return x / checked.y;
}

int main() {
  bar(3, 0); 
}
```

### Example 1
- CT = `constexpr`-context
```cpp
constexpr int foo(int x) {
	return x * x;
}

static_assert(foo(2) == 4);  // CT

int main() {
	constexpr int x = 10;  // CT
	std::size_t y = 10;  // RT / CT
	std::cin >> y;
	int z = foo(y);
	int t = foo(10);

	return 0;
}
```
- Переменная `x` в листинге выше по идее лежит на стеке, однако стек - RT
- На самом деле, как правило он просто подставляет

```cpp
#include <iostream>

constexpr int foo(int x) { return x * x; }

int main() {
	constexpr int x = foo(4);
	std::cout << x << '\n';
}
```
- Здесь будет тупая подстановка `16`
- Если брать адрес `&x`, то это будет адрес ROW-DATA

### Example 2
```cpp
int foo(int N) { std::cout << N; return 0; }

constexpr int bar(int N) {
	return N < 10 ? 9 : foo(N);
}
```
- Это `constepxr`-функция при `N < 10`: путь для компилятора установлен корректно, невзирая на нечистоту `constexpr`-выражения

- Продолжаем разговор
```cpp
// continue

template <int N>
void F()
  requires check<N>
{};

template <int N>
void F() {}

int main() {
  F<5>();    // OK
  F<100>();  // CE
}
```

- Продолжаем разговор
```cpp
// continue
template <int N>
constexpr bool IsValid = N >= 10 || check<N>;

template <int N>
void G() requires IsValid<N> {};

G<100>();  // CE все равно: невзирая на ленивость вычислений, компилятор все равно будет пытаться инстанцировать check<N=100> (вдруг там переопределен operator||).
```

- Продолжаем разговор
```cpp
// continue
template <int N>
constexpr bool IsValid = check<N>;

template <int N>
void G() requires N >= 10 || IsValid<N> {};

G<100>();  // OK
```

# Литеральные типы
- Наконец, вводим понятие литерального типа

- _**Def**_ Тип является литеральным если:
	1. У него есть `constexpr` деструктор
	2. У него есть `constexpr` конструктор (причем это не `copy`- и не `move`- конструктор)

```cpp
class Point {
public:
    constexpr Point(double x, double y) : x_(x), y_(y) {}
    constexpr double getX() const { return x_; }
    constexpr double getY() const { return y_; }

public:
    double x_;
    double y_;
};

int main() {
    constexpr Point p(10.5, 20.5);
    static_assert(p.getX() == 10.5, "X coordinate should be 10.5");
    static_assert(p.getY() == 20.5, "Y coordinate should be 20.5");
}
```

# New в compile-time
- Вектор с C++20 является литеральным типом. А значит, в compile-time как-то поддержали аллокацию.
```cpp
// При этом в global scope или в main можно создать только пустой вектор
constexpr std::vector<int> vec1;  // OK
constexpr std::vector<int> vec2 = {1, 2, 3};  // CE
```

- Действительно, в C++20 разрешили выделять память в compile-time (`new`, `delete`)
- Требования следующие: вы обязаны деаллоцировать всю память после выхода из главной внешней функции. Пример:

```cpp
constexpr int* GetInt(int value) {
    return new int(value);
}

constexpr int foo() {
    auto new_value = GetInt(10);
    delete new_value;
    return 10;
}

constexpr int test = foo();
```

```cpp
constexpr int bar() {
	int* ptr = new int[10];
	for (std::size_t i = 0; i < 10; ++i) {
		ptr[i] = i;
	}
	std::site_t res = 0;
	for (std::size_t i = 0; i < 10; ++i) {
		res += ptr[i];
	}
	delete[] ptr;
	return res;
}

static_assert(bar() == 45);

int main() {
	constexpr int x = foo(10);
}
```

# Виртуальные функции в compile-time
- Начиная с C++20 в compile-time можно использовать виртуальные функции (но не виртуальное наследование)

```cpp
struct Base {
    constexpr Base() = default;
    constexpr virtual ~Base() = default;

    constexpr virtual int foo() { return 1; }
};

struct Derived: public Base {
    constexpr Derived() = default;
    constexpr ~Derived() override = default;

    constexpr int foo() override { return 2; }
};

constexpr int foo() {
    Derived d;
    Base& b = d;
    return b.foo();
}

static_assert(foo() == 2);
```

# Consteval и constinit

### `consteval`
- `constexpr`-функция предполагает, что она может вычислиться в runtime. Поэтому был придуман функционал для явного указания требования на вычисление в compile-time.
- Чтобы гарантировать, что функция вызовется именно в compile-time, нужно явно сказать компилятору об этом: функции, помеченные `consteval`, обязаны выполнятся только на этапе компиляции

```cpp
consteval int foo(int n) { return n * n; }

int main() {
	constexpr int y = foo(10); // OK

	int x = 10;
	int xx = foo(x); // CE

	int xxx = foo(10); // OK - CT-контекст виден
}
```

#### Example
```cpp
// Функция **обязана** быть вычислена в compile-time
consteval int f(int x) {
	return x + x;
}
```
- Настолько круто, что ее не существует в RT (т.е. в коде программы) -> у нее даже нельзя взять адрес

### `constinit`
- С `constinit` все сложней: этот модификатор указывает, что переменная должна быть инициализирована на этапе компиляции. При этом эту переменную можно менять.

> The `constinit` specifier declares a variable with static or thread storage duration.

- То есть можно объявлять только статические и глобальные переменные.

```cpp
#include <iostream>
#include <string>

struct LoggerConfig {
    int logLevel;
    std::string logPath;
};

// constinit указывает, что инициализация должна произойти на этапе старта программы
constinit LoggerConfig globalLoggerConfig{3, "/var/log/myapp.log"};

int main() {
    // при запуске программы globalLoggerConfig уже инициализирован
    std::cout << "Log Level: " << globalLoggerConfig.logLevel << std::endl;
    std::cout << "Log Path: " << globalLoggerConfig.logPath << std::endl;

    // так как это constinit, мы можем изменять значения после инициализации
    globalLoggerConfig.logLevel = 4; // Допустимо

    std::cout << "Updated Log Level: " << globalLoggerConfig.logLevel << std::endl;

    // однако, следующий код не скомпилируется, если globalLoggerConfig был объявлен как constexpr
    // constexpr LoggerConfig testConfig{1, "/test.log"};
    // testConfig.logLevel = 2; // ошибка компиляции, т.к. constexpr не допускает изменений после инициализации
    return 0;
}
```

#### Example
```cpp
constexpr int foo(int x) { return x * x; }

// Можно только глобальные и static
constinit int x = foo(100);  // Обязана честно инициализироваться

int main() {
	constinit int x = foo(100);  // CE: Запрещено использовать во фреймах функций
}
```

# `constexpr` переменные внутри `constexpr` функций

- Рассмотрим пример
```cpp
template <typename T>
consteval size_t foo(std::initializer_list<T> init) {
    constexpr size_t init_size = init.size();  // CE
    return init_size;
}

constexpr size_t n = foo({1, 2, 3});  // CE
```
- CE, потому что переменная `init` не является `constexpr` (и не может, так как это параметр функции)

- Однако если убрать `constexpr`, все заработает
```cpp
template <typename T>
consteval size_t foo(std::initializer_list<T> init) {
    size_t init_size = init.size();
    init_size += 1;  // Даже можно менять
    return init_size;
}

constexpr size_t n = foo({1, 2, 3}); // OK
```

- То есть переменная `init_size` является `constexpr`, но правилами языка запрещено объявлять её таковой. Более того, мы получили `constexpr`-переменную, которую можно менять

# Честный разговор про constant-evaluation
```cpp
int foo(int a, int b) {
	constexpr int z = blabla(); // тут рекурсивно запуститься новый интерпретатор => своя память должна быть очищена на этой же строчке
	int y = 10;
	if (y + a > 10) {
		bar(x) {
			baz() {
				
			}
		}
	}
	// delete всего должен быть здесь (при выходе из всей "пещеры", по завершении работы интерпретатора)
}

constexpr int x = foo(2, 3) /* на этом моменте начинается const evaluation - здесь стартует интерпретатор */;
```

- Повышаем градус
```cpp
foo(constexpr int& x) { // ссылка константная => side-эффекта не будет
}

// может вызываться только для того, что создано интерпретатором
constexpr foo(int& x) {
	++x;
}

constexpr bar() {
	int x = 0;
	foo(x);
}
```

- По идее `constexpr`-функции кэшируются (ведь при одинаковых аргументах => одинаковое возращаемое значение)

- Способ обойти кэширование - наши любимые лямбды!
```cpp
template <auto V = [](){}>
constexpr bar() { ... }
```

### Example 1
```cpp
#include <iostream>

const int x = 10;  // могут использоваться как CT - интегральный тип
constexpr int y = x + 3;  // вообще CT

static_assert(y == 13);

consteval int foo(std::vector<int> vec) {
	// constexpr int sz = vec.size();  // CE - vec is not constant expression (должно быть проинициализировано константным выражением)
	std::size_t sz = vec.size();  // OK
	return sz + 3;
}

constexpr int enter(int x) {
	std::vector<int> v(x, 0);
	return foo(v);
}

static_assert(enter(10) == 13);

```

### Example 2
- То, что известно в compile-time, и то, что `constexpr`, - это разное
```cpp
template <std::size_t I>
struct Predicate {
	constexpr bool operator()(int x) { return x > I; }
};

template <typename Pred>
constexpr auto call(Pred pred, int x) {
	// it's not a constant expression (pred is not an constant)
	if constexpr (pred(x)) {
		return 10;
	} else {
		return 2.2;  // Это нормально - constexpr же???
	}
}

constexpr auto x = call(Predicate<10>{}, 0);  // CE

int main() {
	std::cout << x;
}
```

### Example 3
- Улучшим `std::advance` для итераторов
```cpp
#include <iterator>

template <typename Iterator>
void my_advance(Iterator& iter, int n) {
  if constexpr (std::is_same_v<typename std::iterator_traits<Iterator>::iterator_category, std::random_access_iterator_tag>) {
    iter += n;
  } else {
    for (int i = 0; i < n; ++i) {
      ++iter;
    }
  }
}
```
