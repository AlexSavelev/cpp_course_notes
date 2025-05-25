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

}
```

Вычисления на шаблонах - тьюринг полны. Но есть некоторые недостатки:
- Можем использовать ограниченний функционал: рекурсия, ифы
- Не можем нормально эмулировать циклы - ограничены глубиной рекурсии
- Код не интуитивный

# Constexpr вычисления 
```cpp
constexpr int factorial(int n, int mod) {
    return N == 0 ? 1 : factorial(n-1, mod) * n % mod;
}

static_assert(factorial(1,1) == 1);
```

- Добавили к функции модификатор constexpr
- В 11 стандарте это уже работает. Но можно писать только return что-то. Строчка может быть сколь угодно длинная, но должна быть одна
- В 11 стандарте это работало как синтаксический сахар - все превращалось в шаблоны

Начиная с 14го стандарта механизм constexpr функций сильно расширили. Теперь факториал можно считать вот так:

```cpp
constexpr int factorial(int n, int mod) {
    int result = 1;
    for (int i = 1; i <= n; ++i) {
        result *= i;
        result *= mod;
    }
    return result;
}
```

- Грубо говоря: завезли интерпретатор внутрь компилятора
- Этот процесс называется constant evaluation - отдельный этап



==TODO== about static: initialize non-const in behind of class decl

Новый этап const-evalualion. Теперь есть интерпретатор, вычисл.
==TODO== asan
В compile-time мы видим весь flow наших функций. В runtime - нет. Это главная проблема


```cpp
int foo(int N) { std::cout << N; return 0; }

constexpr int bar(int N) {
	return N < 10 ? 9 : foo(N);
}
```
- Это `constepxr`-функция при `N < 10`: путь для компилятора установлен корректно, невзирая на нечистоту `constexpr`-выражения

```cpp
// continue
template <int N>
constexpr bool check = bar(N) < 10;

template <int N>
void F() requires check<N> {};

template <int N>  // TODO req?
void F() {}  // TODO req?

F<5>();  // OK
F<100>();  // CE
```

```cpp
// continue
template <int N>
constexpr bool IsValid = N >= 10 || check<N>;

template <int N>
void G() requires IsValid<N> {};

G<100>();  // CE все равно: невзирая на ленивость вычислений, компилятор все равно будет пытаться инстанцировать check<N=100> (вдруг там переопределен operator||).
```

```cpp
// continue
template <int N>
constexpr bool IsValid = check<N>;

template <int N>
void G() requires N >= 10 || IsValid<N> {};

G<100>();  // OK
```

==TODO== lecture
==TODO== constexpr-specifier in CPPREF
==TODO== undefined behaviour allowed in constexpr STACKOVERFLOW


- ==TODO== checkout 
- ==TODO== all CProject code listings
# Sem ~30

- `constexpr`:
	- Для функций - лишь указание, что ф-ция мб вычислена в compile-time (CT)
	- Для переменный - объявляем CT

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

- CT = constexpr context

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

#### Constinit
```cpp
constexpr int foo(int x) { return x * x; }

// Можно только глобальные и static
constinit x = foo(100);  // Обязана честно инициализироваться

int main() {
	constinit x = foo(100);  // Запрещено использовать во фреймах функцийы
}

```

#### Consteval
```cpp

// Ф-ция **обязана** вычислена в CT
consteval int f(int x) {
	return x + x;
}

```
- Настолько круто, что ее не существует в RT - у нее даже нельзя взять адрес

### More about constexpr
```cpp

// В C++11 constexpr функция могла быть только одна строчка - с return 
constexpr int foo11() {
	return 1;
}

// В C++14 появился процесс "constant evaluation" - исполняется интерпретатором
constexpr int foo14() {
	if (x > 2) {
		return x * x;
	}
	return 1;
}

```
- Про C++14
- Понятно, что нельзя вызывать RT-объекты: `std::cout`
```cpp
constexpr int foo14() {
	if (x > 2) {
		return x * x;
	}
	std::cout << x;
	return 1;
}
```
- При этом `foo14(10)` - OK - вычисляется интерпретатором сверху-вниз
- `foo14(0)` - CE

- С C++23 вроде можно писать RT-инструкции - компилятор будет игнорить

### Constant evaluation
- Сейчас constexpr может быть все, что угодно

Literal type - определение
1) Все поля - литеральные
2) `constexpr` `~Literal()`
3) `constexpr` constructor (хотя бы один, причем не copy- и не move- конструктор)


```cpp
#include <iostream>

class Literal {
	Literal() {}
	constexpr Literal(int) {}
	constexpr Literal(const Literal& other) {}
	constexpr ~Literal() {}
};

constexpr Literal t = Literal{11};

constexpr int foo(int x) {
	Literal lit{11};
}
```
==TODO== const getters or constexpr??

- С C++20 разрешили выделять память в CT (`new`, `delete`)
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

- Про вектор
```cpp
#include <iostream>

// Пустой может, такой не может создать в CT
constexpr std::vector<int> vec = {1, 2, 3};
```

- Честный разговор про const evaluation
```cpp
int foo(a, b) {
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

constexpr int x = foo(2, 3) /* на этом моменте начинается const eval - здесь стартует интерпретатор */;
```

- Повышаем градус
```cpp
foo(constexpr int& x) { // ссылка константная => side-эффекта не будет
	foo(x);
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

- По идее constexpr функции кэшируются (ведь при одинаковых аргументах => одинаковое возращаемое значение)

- Способ обойти кэширование - наши любимые лямбды!
```cpp
template <auto V = [](){}>
constexpr bar() { ... }
```

#### constant expression
```cpp
#include <iostream>

const int x = 10;  // могут использоваться как CT - интегральный тип
constexpr int y = x + 3;  // вообще CT

static_assert(y == 13);

consteval int foo(std::vector<int> vec) {
	constexpr int sz = vec.size();  // CE - vec is not constant expression (должно быть проинициализировано константным выражением)
	// OK:
	// std::size_t sz = vec.size();
	return sz + 3;
}

constexpr int enter(int x) {
	std::vector<int> v(x, 0);
	return foo(v);
}

static_assert(enter(10) == 13);

```

```cpp
template <std::size_t I>
struct Predicate {
	constexpr bool operator()(int x) { return x > I; }
};

template <typename Pred>
constexpr auto call(Pred pred, int x) {
	if constexpr (pred(x)) {  // it's not a constant expression (pred is not an constant...????)
		return 10;
	} else {
		return 2.2;  // Это нормально - constexpr же
	}
}

constexpr auto x = call(Predicate<10>{}, 0);  // CE

```

- То, что известно в CT, и то, что constexpr, - это разное

#### exception
- const evaluation выполняется интерпретатором => почему бы не вызывать `throw()` ?
- `catch` запрещен
- Но все равно хороший способ ловить ошибки

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
struct CheckIsNotZero {
	consteval CheckIsNotZero(int y) {
		if (y == 0) {
			throw 
		}
	}

	consteval Check(int y) {
		if (y == 0) {
			throw 
		}
	}
};

int foo(int x, int y) {
	
}

int main() {
	foo(10);
}
```
==TODO== listing
