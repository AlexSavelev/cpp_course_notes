Copy elision is a compiler optimization technique that eliminates unnecessary copying/moving of objects.
# RVO, NRVO
- _**Def:**_ RVO - Return Value Optimization
- _**Def:**_ NRVO - Named Return Value Optimization
- [Habr article](https://habr.com/ru/companies/vk/articles/666330/)

### NRVO
- По идее, при вызове функции `f()` создастся локальная переменная `local_var`, затем создасться переменная-результат `result` в виде копии `local_var`. При выходе из функции со стека `local_var` снимается.
- При NRVO же вместо создания `local_var` компилятор сразу создаст  `result` конструктором по умолчанию в точке вызова функции `f()`. А функция `f()` будет выполнять действия сразу с переменной `result`. То есть в этом случае не будет вызван ни конструктор копии, чтобы скопировать `local_var` в `result`, ни деструктор `local_var`.
- Технически это выглядит так:
	1. Компилятор создаёт конструктором по умолчанию до вызова функции `f()` переменную `result`
	2. Затем неявно передаёт в функцию `f()` указатель на `result`
	3. В рамках функции `f()` не создаёт `local_var`, а вместо этого работает с указателем на `result`
	4. в `return` ничего не копируется, поскольку данные уже там

```cpp
#include <iostream>
#include "Verbose.hpp"

Verbose f() {
  std::cout << "1\n";
  Verbose local_var;
  std::cout << "2\n";
  return local_var;
}

int main() {
  std::cout << "0\n";
  auto result = f();
  std::cout << "3\n";
}

```

#### Output with NRVO:
```OUT
0
1
Verbose 1 default constructed
2
3
Verbose 1 destructed
```

### RVO
- RVO - это частный случай NRVO, когда экземпляр возвращаемого класса создаётся прямо в операторе `return`
```cpp
#include <iostream>
#include "Verbose.hpp"

Verbose f() {
  std::cout << "1\n";
  return Verbose();
}

int main() {
  std::cout << "0\n";
  auto result = f();
  std::cout << "2\n";
}

```

#### Output with RVO:
```OUT
0
1
Verbose 1 default constructed
2
Verbose 1 destructed

```

==TODO== [ST](https://stackoverflow.com/questions/12953127/what-are-copy-elision-and-return-value-optimization)


