# База
## Resources
- cppreference
- stackoverflow
- google
- google C++ codestyle
- godbolt.org
- Курс Мещерина ФПМИ
## Compilers
- G++ (g++ main.cpp -o main -> ./main)
- ***Clang***
- etc
## Характеристики
- Компилируемый
- Статическая типизация
- Слабая типизация

## `int main`
- Точка входа в программу

## Литералы
- `5` - `int`
- `5.0` - `double`
- `5.0f` - `float`
- `'a'` - `char`
- `"aaa"` - `const char*`
- `true` - `bool` + keyword

## Идентификаторы
- `x`
- `y`
- `make_array`

## Branches & loops

```cpp
if (bool-expr) statement;

if (init-statement; bool-expr) { statement; } // from C++17

while (bool-expr) { statement; }

do { statement; } while (bool-expr);

for (init-statement; bool-expr; expr) { statement; }
```

## Тернарный оператор

- Syntax: `a ? b : c;`
- Example: `x = (x < 2 : 1 : 2);`
- Note: Возращаемые значения должны быть одного типа

