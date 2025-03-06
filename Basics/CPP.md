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

## `switch`
```cpp
switch (init-statement; condition) {  // C++17
	case expr1:
		// ...
		break;
	case expr2:
		// ...
		break;
	// ...
	default:
		// ...
}
```
- В качетсве `condition` могут быть:
	- integral types
	- enumeration types
	- class types
		- If the yielded value is of a class type, it is contextually implicitly converted to an integral or enumeration type.
- Если не ставить `break;` после каждого `case`'а, программа будет проваливаться ниже
- Начиная с C++17 при таких обстоятельствах (переходы между `case` без `break`) компилятор будет выдавать Warning'и
	- Чтобы вернуть былую фичу, необходимо указать аттрибут `[[fallthrough]]`
```cpp
void f(int n) {
    void g(), h(), i();
 
    switch (n) {
        case 1:
        case 2:
            g();
            [[fallthrough]];
        case 3: // no warning on fallthrough
            h();
        case 4: // compiler may warn on fallthrough
            if (n < 3) {
                i();
                [[fallthrough]]; // OK
            }
            else {
                return;
            }
        case 5:
            while (false) {
                [[fallthrough]]; // ill-formed: next statement is not
                                 //             part of the same iteration
            }
        case 6:
            [[fallthrough]]; // ill-formed, no subsequent case or default label
    }
}
```

## Тернарный оператор

- Syntax: `a ? b : c;`
- Example: `x = (x < 2 : 1 : 2);`
- Note: Возращаемые значения должны быть одного типа

