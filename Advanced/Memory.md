# Memory types
### Kernel space
### Stack
- локальные переменные и адреса возвратов
### Memory mapping Segment
- file mappings: DLL, .so
### Heap
### BSS segment
- uninitialized static vars: `static char* username;`  ==TODO==
### Data segment
- static vars inst. by program: `static char* a = "hello";` (статические и глобальные переменные, некоторые константы)
### Text segment (ELF)
- bin image of the process (машинный код)


# About stack
- Local variable кладется в стек при ее создании, при выходе из области видимости (scope'а) переменная снимается со стека
- На стек кладутся адреса возвратов при заходе в функции
- stack overflow - переполнение стека

# Dynamic memory
- Иначе говоря, куча
- operator new
	- `int* a = new int(10);`
- operator delete
	- `delete a;`
- memory leak - утечка
	- `int* a = new int(5);`
	- `a = new int(10); // забыли старый адрес`
	- `delete a;`
- double free
	- `int a = new int(5);`
	- `delete a;`
	- `delete a; // после первого вызова delete память уже не наша`
- `operator new[]` & `operator delete[]`
	- `int* ptr = new int[10];`
	- `delete[] ptr;`
- C-style arrays
	- `int arr[10]; // sizeof(arr) = 40`
	
`std::addressof(a)` ==TODO==
# `new`, `delete` overloading

```cpp
void* operator new(size_t n) {
	void* ptr = malloc(n);
	if (ptr == nullptr) { throw std::bad_alloc(); }
	return ptr;
}

void operator delete(void* p) noexcept {
	free(p);
}
```
==TODO== why noexcept
_**Note**_ Перегружать глобальный `new` - плохо

### `new` full implementation

```cpp
void* p = malloc(sizeof(T) * n)  // + align (because of C++ STD)
while (p == std::nullptr && new_handler) {
	new_handler();
	p = malloc(sizeof(T) * n;
}
if (p == std::nullptr) {
	throw std::bad_alloc();
}
// call T() for each element
return reinterpret_cast<T*>(p);
```

- `new_handler` - функция или функтор, призванная найти каким-то образом память (путем очистки указателей и тд.) (clearing an in-memory cache, or destroying some objects that are no longer needed)
- `std::get_new_handler()`, `std::set_new_handler(f)`
- [more info & pseudo code](https://stackoverflow.com/questions/28824317/operator-new-new-handler-function-in-c)
- [also more info](https://stackoverflow.com/questions/18945537/how-does-stdset-new-handler-make-more-memory-available)

### `new_handler()` implementation

Варианты реализации `new_handler()` (как завершать процесс):
1) `throw`
2) `set_new_handler(nullptr);`
3) `set_new_handler(f, 3);`  // счетчик

#### Кастомный `new_handler` в классах
```cpp
template <typename T>
struct Base {
	static void* operator new(std::size_t count, std::align_val_t al) {
		handler = set_new_handler(handler); // swap handlers (now handler is old one)
		void* p = ::operator new(count); // cast to global scope
		// checking out: ptr is valid
		handler = set_new_handler(handler); // return to old good global handler
		return p;
	}

	static void operator delete(void* p) { ... }

	static new_handler_t handler = get_new_handler();
};

struct S : Base<struct STag> {...};
struct S1 : Base<struct S1Tag> {...};
```

- Зачем `Base`?
	- Мы хотим сделать интерфейс уникального `new_handler` для произвольного класса
	- Основная идея: наследование
	- Возможно, можно макросами (`#define CUSTOM_NEW_HANDLER private:static ...`)
- Зачем `template`?
	- Без шаблонов и у `S`, и у `S1` будут общий `static handler field`
	- Но родительский шаблон как ребенок (`S : Base<S>`) - это какой-то кринж, поэтому делаем заглушку `Tag`, объявленную в `param list`

### Перегрузка для конкретного типа
```cpp
struct S {
	static void* operator new(size_t n) { ... }
	static void operator delete(void* p) { ... }
};
```

_**Note**_ Статик будет по умолчанию, но лучше не забывать писать
_**Note**_ При выборе, какой оператор выбрать для типа, предпочтение будет отдано оператору, который объявлен внутри класса

### Placement new
- На выделенной памяти вызывает конструктор
- Если для типа определен кастомный оператор `new`/`delete` то placement new сгенерирован не будет

```cpp
#include <memory>

S* p = reinterpret_cast<S*>(operator new(sizeof(S))); // global new
new(p) S();
operator delete(p);
```

# Allocators
- see [[Allocators]] page
