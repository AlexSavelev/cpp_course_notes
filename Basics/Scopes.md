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

### Definitions
In informal terms, a _nested-name-specifier_ is the part of the _id_ that:
- begins either at the very beginning of a _qualified-id_ or after the initial scope resolution operator (`::`) if one appears at the very beginning of the _id_ and
- ends with the last scope resolution operator in the _qualified-id_.

Very informally, an _id_ is either a _qualified-id_ or an _unqualified-id_. If the _id_ is a _qualified-id_, it is actually composed of two parts: a nested-name specifier followed by an _unqualified-id_.

### Example
```cpp
struct A {
    struct B {
        void F();
    };
};
```
- `A` is an _unqualified-id_.
- `::A` is a _qualified-id_ but has no _nested-name-specifier_.
- `A::B` is a _qualified-id_ and `A::` is a _nested-name-specifier_.
- `::A::B` is a _qualified-id_ and `A::` is a _nested-name-specifier_.
- `A::B::F` is a _qualified-id_ and both `B::` and `A::B::` are _nested-name-specifiers_.
- `::A::B::F` is a _qualified-id_ and both `B::` and `A::B::` are _nested-name-specifiers_.

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
