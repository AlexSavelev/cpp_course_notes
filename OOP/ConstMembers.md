# const-method
- Метод, который можно вызвать у константного объекта
	- Нельзя менять поля объекта
	- Нельзя вызывать non-const methods
- `mutable` - ключевое слово, дающее возможность поменять поле в константном объекте

```cpp
struct T {
	int a = 10;

	void foo() const {
		std::cout << a;
	}
};

int main() {
	const T t;
	t.foo(); // OK
}
```

### Перегрузка по константности
```cpp
struct A {
	void foo() { std::cout << 1; }
	void foo() const { std::cout << 2; }
};

int main() {
	const A a;
	a.foo(); // 2
}
```

# Навешивание
- Константность на поля класса навешивается "справа":
```cpp
struct {
	const int* ptr;
	int& a;

	void foo() const {
		*ptr = 20; // OK
		a = 5; // OK
	}
};
```
