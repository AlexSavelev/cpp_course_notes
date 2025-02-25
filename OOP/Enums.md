# Bad code (C-style):
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

# Good code (C++):
```cpp
enum class Color {
	RED = 0,
	GREEN,  // 1
	BLUE  // 2
};

Color a = Color::BLUE;
```
# Choose type of consts in enums
```cpp
enum class Color : uint8_t {
	RED,
	GREEN,
	BLUE
};
```
