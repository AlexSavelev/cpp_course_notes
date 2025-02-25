# Global scope

```cpp
int x = 3;

int main() {
	int x = 5;
	{
		int x = 2;
		std::cout << ::x;  // ::x - global variable
	}
}
```

# Qualified ID

```cpp
namespace N {
	int f(int x);
}
int N::f(int x) { // Qualified ID
	...
}
```

==TODO==https://stackoverflow.com/questions/4103756/what-is-a-nested-name-specifier
https://stackoverflow.com/questions/7257563/what-are-qualified-id-name-and-unqualified-id-name
# Namespaces
- `using` - механизм для более удобного обращения к неймспейсам
- Также можно делать псевдонимы
```cpp
namespace some_long_name {
  namespace another_name {
    int foo() {
      return 1;
    }
  }
}
namespace al = some_long_name::another_name;

int main() {
  std::cout << al::foo();
}
```
