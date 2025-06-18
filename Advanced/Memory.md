# Memory types
### Kernel space
### Stack
- локальные переменные и адреса возвратов
### Memory mapping Segment
- file mappings: DLL, .so
### Heap
### BSS segment
- BSS - block starting symbol
- uninitialized static vars: `static char* username;`
### Data segment
- static vars inst. by program: `static char* a = "hello";` (статические и глобальные переменные, некоторые константы)
### Text segment (ELF)
- bin image of the process (машинный код)


# About stack
- Local variable кладется в стек при ее создании, при выходе из области видимости (scope'а) переменная снимается со стека
- На стек кладутся адреса возвратов при заходе в функции
- stack overflow - переполнение стека

# About BSS
- BSS - block starting symbol
	- Joke - Better Save Space

### Зачем нужен BSS
The reason is to reduce program size. Imagine that your C program runs on an embedded system, where the code and all constants are saved in true ROM (flash memory). In such systems, an initial "copy-down" must be executed to set all static storage duration objects, before `main()` is called. It will typically go like this pseudo:
```cpp
for(i=0; i<all_explicitly_initialized_objects; i++)
{
  .data[i] = init_value[i];
}

memset(.bss, 
       0, 
       all_implicitly_initialized_objects);
```
Where `.data` and `.bss` are stored in RAM, but `init_value` is stored in ROM. If it had been one segment, then the ROM had to be filled up with a lot of zeroes, increasing ROM size significantly.

RAM-based executables work similarly, though of course they have no true ROM.

Also, `memset` is likely some very efficient inline assembler, meaning that the startup copy-down can be executed faster.

- Иначе говоря, .bss - это некого рода оптимизация, делающая исполняемые файлы меньше и быстрее загружаемыми
- [Source](https://stackoverflow.com/questions/9535250/why-is-the-bss-segment-required)

К слову, в книге Джеффа Дантеманна "[Assembly Language Step-by-Step: Programming with Linux](https://jagdishkapadnis.wordpress.com/wp-content/uploads/2015/05/assembly-language-step-by-step-programming-with-linux-3rd-edition.pdf)" секции **.data** и **.bss** соответственно определяются следующим образом:

1. **.data**:
> The **.data** section contains data definitions of initialized data items. Initialized data is data that has a value before the program begins running. These values are part of the executable file. They are loaded into memory when the executable file is loaded into memory for execution.
> 
> The important thing to remember about the .data section is that the more initialized data items you define, the larger the executable file will be, and the longer it will take to load it from disk into memory when you run it.

2. **.bss**
> Not all data items need to have values before the program begins running. When you’re reading data from a disk file, for example, you need to have a place for the data to go after it comes in from disk. Data buffers like that are defined in the **.bss** section of your program. You set aside some number of bytes for a buffer and give the buffer a name, but you don’t say what values are to be present in the buffer.
> 
> There’s a crucial difference between data items defined in the .data section and data items defined in the .bss section: data items in the .data section add to the size of your executable file. Data items in the .bss section do not. A buffer that takes up 16,000 bytes (or more, sometimes much more) can be defined in .bss and add almost nothing (about 50 bytes for the description) to the executable file size.

// TL;DR =)
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

```cpp
#include <iostream>

int main() {
  int* ptr = new int[10]{1, 2, 3, 4, 5, 6};
  for (std::size_t i = 0; i < 10; ++i) {
    std::cout << ptr[i] << '\n';  // 1234560000
  }
  delete[] ptr;
}
```

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
_**Note**_ Перегружать глобальный `new` - плохо

### `new` full implementation

```cpp
void* p = malloc(sizeof(T) * n)  // + align (because of C++ standart)
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
- Имеет синтаксис: `new(ptr) S(args...)`

- Рассмотрим на примере
```cpp
struct S {
 private:
  S() = default;
 public:
};
int main() {
  S* p = reinterpret_cast<S*>(operator new(sizeof(S)));
  new(p) S();  // CE
  operator delete(p);
}
```
- CE, так как конструктор `S` приватный (так и задумано). Если конструктор сделать публичным, все сработает

- Если для типа определен кастомный оператор `new`/`delete` то placement new сгенерирован не будет
```cpp
struct S {
 public:  // даже если сделать публичным
  S() = default;
 public:
  static void* operator new(size_t n) {
    void* ptr = malloc(n);
    if (ptr == nullptr) {
      throw std::bad_alloc();
    }
    return ptr;
  }
  static void operator delete(void* p) { free(p); }
};

int main() {
  S* p = reinterpret_cast<S*>(operator new(sizeof(S)));
  new(p) S(); // CE
  operator delete(p);
}
```

- План таков: перегрузим placement new и сделаем его другом
```cpp
struct S {
  friend void* operator new(size_t, S*);
 private:
  S() = default;
 public:
};

void* operator new(size_t, S* p) {  // size_t = sizeof(S)
  return p;
}

int main() {
  S* p = reinterpret_cast<S*>(operator new(sizeof(S)));
  new(p) S();
  operator delete(p);
}
```
- Перегруженная функция не вызывает конструктор. Соответственно, `friend` ни чего нам не дал

#### Про placement-new
- На этом моменте стоит сделать небольшую аннотацию
- На самом деле все методы класса `S::method(Arg1 arg1, Arg2 arg2, ...)` на стадии компиляции будут "рассахарены" прибавлением в начало списка аргументов `[const] S* this`. Это правило работат в том числе и для конструкторов и деструкторов.
- Placement new же "рассахаривается" следующим образом:
```cpp
new(ptr) S(args...) => S(new(ptr), args...)
```
- Именно поэтому делание `operator new(size_t, S*)` другом не помогло - наш оператор никак не заведует вызовом конструктора. Грубо говоря, конструктор вызывается на строчке, где прописан placement-new
- Таким образом, конструктор в любом случае придется делать публичным

#### Example
- Кастомный оператор `delete` при этом компилятор не вызывает, кроме случая, когда в операторе `new` произойдет исключение
```cpp
#include <iostream>

struct S {
  S() { throw 1; }
};

void* operator new(size_t n, double d) {
  std::cout << "custom new with d: " << d << std::endl;
  return malloc(n);
}

void operator delete(void* p, double d) {
  std::cout << "custom delete with d: " << d << std::endl;
  return free(p);
}

int main() {
  try {
    S* p = new (3.14) S();
  } catch (...) {
  }
}
```
- Как видно, в аргументах оператора placement-new может быть не только указатель
- И именно здесь аллоцируется память

# Allocators
- see [[Allocators]] page
