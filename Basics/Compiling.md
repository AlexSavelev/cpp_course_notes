
# C++ compilation pipeline
```
Preprocessor -> Translation (Compiling) -> Assembling -> Linking
```

# Preprocessor
- Подготовка C++ кода к компиляции
- cpp -> cpp
- [More](https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B5%D0%BF%D1%80%D0%BE%D1%86%D0%B5%D1%81%D1%81%D0%BE%D1%80_%D0%A1%D0%B8)

### Препроцессором выполняются следующие действия
- Замена соответствующих диграфов и триграфов на эквивалентные символы «#» и «\»
- Удаление экранированных символов перевода строки
- Замена строчных и блочных комментариев пустыми строками (с удалением окружающих пробелов и символов табуляции)
- Вставка (включение) содержимого произвольного файла (`#include`)
- Макроподстановки (`#define`)
- Условная компиляция (`#if`, `#ifdef`, `#elif`, `#else`, `#endif`)
- Вывод сообщений (`#warning`, `#error`)


### G++ preprocessor only
- Using flag `-E`
`g++ -E main.cpp > main_pr.cpp`
`g++ -E -D<DEFINE_EXC=VAL> main.cpp > main_pr.cpp`


# Translation (compiling)
- Этап перевода C++ кода на язык ассемблера
- cpp -> asm
- Осуществляется манглирование имен, инстанцирование шаблонов

### Name mangling

| C++         | ASM    |
| ----------- | ------ |
| foo(int)    | fooi() |
| foo(double) | food() |
- [More about name mangling](https://en.wikipedia.org/wiki/Name_mangling)

### G++ compiling only
- Using flag `-S`
`g++ -S main.cpp -o main.s`


# Assembling
- asm -> obj

### G++ assembling only
- Using flag `-c`
`g++ main.cpp -c prog`
- Run: `./prog`

# Linking
- Links object files
### G++ linking
`g++ a.o b.o main.o -o prog` or `g++ build/* -o prog`
- Run: `./a.out`

# G++ flags
- Include additional directories:
	- `-I include/`
	- `g++ -I include/ main.cpp -c -o build/prog`

# Keywords

### `extern`
- Show compiler that definition is located in other `.cpp` file or `.c` file
	- Other `.cpp` file: `extern int foo(int, int);`
	- In `.c` file (compatibility): `extern "C" { int foo(int); }`

### `static`
- Static functions can't be called outside the current `.cpp` file
- Alternative: `namespace { ... }` - anonymous namespace



==TODO== static&dynamic libs
