# Bad code (C-style) (unscoped enumeration):
```c
enum Color {
	RED = 0,
	GREEN,  // 1
	BLUE  // 2
};

Color a = BLUE; // It's a global scope const
```
- Обычный `enum` вносит все свои константы в глобальную область видимости
- Это не очень хорошо, поэтому в C++11 ввели `enum class`

# Good code (C++) (scoped enumeration):
- Syntax: `enum class|struct EnumName { ... };`
```cpp
enum class Color {
	RED = 0,
	GREEN,  // 1
	BLUE  // 2
};

Color a = Color::BLUE;
```

# Choose type of consts in enums
- Называются также строго типизированными, или типобезопасными, перечислениями
```cpp
enum class Color : uint8_t {
	RED,
	GREEN,
	BLUE
};
```

# Typeless enums
- По умолчанию базовый тип для enum'ов - `int`
- Однако он может расширяться, если значение превосходит максимальное/минимальное значение `int`'а:

```cpp
#include <cstdint>
#include <iostream>

enum A { RED = INT64_MAX };

int main() {
  std::cout << RED << ' ' << sizeof(A);  // honest INT64_MAX, 8
  return 0;
}
```
