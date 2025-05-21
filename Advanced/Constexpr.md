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

