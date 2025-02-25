# Problem overview
- Проанализируем функцию:
```cpp
template <typename T>
void swap(T& first, T& second) {
  T tmp = first;
  first = second;
  second = tmp;
}
```

- В функции проблема - каждая из 3-х строчек - это копирование
- Или еще: метод `vector::push_back` вызывает конструктор копирования (даже если имеем дело с временным объектом)
	- `emplace_back(const Args... args)` решает проблему, конструируя `T` на основе списка аргументов
	- Как неплохой обходной путь сойдет, но не во всех случаях
- В методе `vector::resize` осуществляется `std::unitialized_copy(...)`, а затем снова копирование
- Но мы хотим просто отдавать _право собственности_

# `std::move`
```cpp
template <typename T>
void swap(T& first, T& second) {
  T tmp = std::move(first);
  first = std::move(second);
  second = std::move(tmp);
}
```

- Перекладываем все ресурсы из объекта в объект, поэтому все вычисления осуществляются за `O(1)`
- То есть с помощью `std::move` мы перекладываем ресурсы

- Рассмотрим `T tmp = std::move(first);`
	- Если просто написать `std::move(first)`, то ничего не произойдет
	- Все стандартные типы гарантируют, что после `auto x = std::move(y)`, `y` остается валидным (пустым почти наверное).

# Move-конструкторы
```cpp
String(String&& s): str_(s.str_), size_(s.size_) {
	s.str_ = nullptr;
    s.size_ = 0;
}
```
- `String(String&& s)` - конструктор с `rvalue` ссылкой на объект типа `String`
- `str_(s.str_), size_(s.size_)` - "забираем" ресурсы у донора
- `s.str_ = nullptr;` - нужно занулить указатель у себя чтобы не было двойного владения (и double free)
	- _**Напоминание:**_ `double free` - это не исключение, это сразу аборт!
- `s.size_ = 0;` - нужно оставить объект валидным и пустым

# Move `operator=`
```cpp
String operator=(String&& s) {
  String tmp = std::move(s); // Пользуемся написанным move-конструктором
  swap(tmp); // Придется написать отдельно
  reutrn *this;
}
```

- `std::move(T)`, где `T` - базовый тип (`int`, `pointer`, ...), просто копирует данные

# Автоматическая генерация и правило пяти
Move-конструктор генерируется автоматически (он будет втупую мувать все поля), если нет самого user-declared move-конструктора и:
1. Нет user-declared copy-конструкторов
2. Нет user-declared операторов присваивания копированием
3. Нет user-declared move-операторов копирования
4. Нет user-declared деструктора

- Компилятор генерирует методы, следуя ему
- С move-оператором присваивания аналогично

### Define
- Обозначения: CC, CO, MC, MO, D
	- C - copy; M - move; D - destructor
	- C - constructor; O - operator
	- Лучше выучить к экзамену

| I will define | Generated automatically | Deprecated | Not generating |
| ------------- | ----------------------- | ---------- | -------------- |
| CC            | D                       | CO         | MC, MO         |
| CO            | D                       | CC         | MC, MO         |
| MO            | D                       | -          | CC, CO, MC     |
| MC            | D                       | -          | CC, CO, MO     |
| D             | CC, CO                  | -          | MO, MC         |

### Почему чаще всего automatic move-конструктор не есть хорошо
- Так сгенериуется `String(String&&)` автоматически:
```cpp
class String {
 public:
  String(String&& s) {
    s.str_ = std::move(s.str_);
    s.size_ = std::move(s.size_);
  }
 private:
  char* str_ = nullptr;
  size_t size_ = 0;
};
```

- `std::move` от тривиальных объектов просто их копирует. То есть мы не обнулим указатель и размер у строки `s`
- Рано или поздно произойдет `double free`

# Value categories. True lvalue, rvalue definition
- Старые определения (lvalue - что может стоять слева, rvalue - все остальное), что вводили, неверные
	- Пример в одну сторону: объект, у которого не определен оператор присваивания, не может стоять слева от `=`, хотя он lvalue
	- Пример в другую сторону: `BitReference` в `vector<bool>`. Это rvalue (временно созданный объект) однако он стоял слева от `=`

- Value-category это характеристика ВЫРАЖЕНИЯ, а не чего-либо еще
![[expression_tree.png]]

_**Def**_: lvalue - это:
1. Идентификаторы
2. Вызов функции, возвращаемый тип которой это lvalue-ссылка
3. и тд...
- Общий смысл: что-то "постоянное", у чего есть имя

_**Def**_: prvalue - это:
1. Литералы (кроме `const char*`)
2. Вызов функции возвращаемый тип которой это `non-reference`
3. и тд...
- Общий смысл: что-то "временное", у чего нет имени
- pure rvalue

_**Def**_: xvalue - это:
1. Вызов функции, тип которой это `rvalue-reference`
2. и тд...
- Общий смысл: относится к мувнутым объектам (отсюда и название - expired)

==TODO== have a party: https://en.cppreference.com/w/cpp/language/value_category

## Links
- `int&` is lvalue link
- `int&&` is rvalue link (rvalue link is lvalue expr)
- `std::move` навешивает `&&`, переводя его в rvalue

| Category | Link type |
| -------- | --------- |
| lvalue   | `T&`      |
| xvalue   | `T&&`     |
| prvalue  | `T`       |

```cpp
int&& rf = 3;  // OK
int&& a  = rf; // CE (rf is lvalue expr)

int&& b = move(rf);
int& c = rf;  // OK (lvalue link links to lvalue expr)

const int&& d = rf;  // CE
const int&& d2 = move(rf);  // OK
int&& e = move(d2);  // cast const volatile blablabla

const int& q = move(d2);  // OK
```


`T&& -> const T&`

==TODO==

rvalue vs lvalue
1. 
2. если ф-ция возвращает rvalue ссылку, то ф-ция - lvalue (rvalue) expr

int x = 0;
int&& y = 1; // продление жизни как при const&
y = x; // здесь копия x, вызывается move и меняется временный объект () (y ссылается на свою память) (если бы было y = std::move(x), то мы бы биндили и x бы инкрементнулся)
y += 1;
std::cout << x;


==TODO== add to revome_reference T& and T&& specialization 

int& y = x; // то же самое что и int& y = static_cast<int&>(x); - нового объекта не создается, мы просто сделали ссылку (поэтому std::move не создает копию)


==TODO== ref qualifiers
// был const qualifiers - когда разделяли operator[] для const и non-const objects
