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


# Table of contents

| No  | Article                                             | Difficulty (1-5) |     |     |
| --- | --------------------------------------------------- | ---------------- | --- | --- |
| 1   | [[Data types]]                                      | 1                |     |     |
| 2   | [[Compiling]]                                       | 2                |     |     |
| 3   | [[Errors]]                                          | 2                |     |     |
| 4   | [[Functions]]                                       | 2                |     |     |
| 5   | [[References & pointers]]                           | 2                |     |     |
| 6   | [[Scopes]]                                          | 2                |     |     |
| 7   | [[Operators]]                                       | 2                |     |     |
| 8   | [[OOP Basics]]                                      | 2                |     |     |
| 9   | [[Enums]]                                           | 1                |     |     |
| 10  | [[Member references]]                               | 3                |     |     |
| 11  | [[Const members]]                                   | 2                |     |     |
| 12  | [[Templates]]<br>- [[Overloading & specialization]] | 3                |     |     |
| 13  | [[Inheritance]]<br>-[[Dynamic polymorphism]]        | 3                |     |     |
| 14  | [[Lambda functions]]                                | 2                |     |     |
| 15  | [[Iterators]]                                       | 2                |     |     |
| 16  | [[Exceptions]]                                      | 1                |     |     |
| 17  | [[Containers]]                                      | 2                |     |     |
| 18  | [[Memory]]<br>- [[Allocators]]                      | 3                |     |     |
| 19  | [[Move semantics]]                                  | 4                |     |     |
|     |                                                     |                  |     |     |
|     |                                                     |                  |     |     |
|     | ==TODO== Misha's Lecture                            |                  |     |     |

#### Additional

| No  | Article         | Difficulty (1-5) |
| --- | --------------- | ---------------- |
| 0   | [[Attachments]] | -                |
| 1   | [[Testing]]     |                  |
| 2   | [[Cringe]]      | 5                |
|     |                 |                  |
