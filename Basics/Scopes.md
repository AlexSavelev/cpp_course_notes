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

# Point of declaration
> The point of declaration for a name is immediately after its complete declarator and before its initializer... `[3.3.2/1]`

```cpp
#include <iostream>

int foo(int* a) {
  *a = 0;
  return 1;
}

int main() {
  int a /*point of declaration*/ = foo(&a);
  std::cout << a << '\n';
  const int x = 4;
  {
    int x[x] /*point of declaration*/ = {1, 2, 3, 4};
    std::cout << "x[2] = " << x[2] << '\n';
  }
}
```

- For example:
```cpp
int x = 101;
{
    int x = x;
    std::cout << x << std::endl;  // garbage
}
```
- Above code is equal to the below one:
```cpp
int x = 101;
{
  int x;
  x = x; // Self assignment, assigns an indeterminate value.
  std::cout << x << std::endl;
}
```
- Because:
```cpp
int x = x; <--// Now, we have the new `x` which hides the older one, 
//   ^        // so it assigns itself to itself
//   |
//   +---// Point of declaration,
//       // here compiler knows everything to declare `x`.
//       // then declares it.
```

- Also, this example shows the point of declaration for an enumerator
```cpp
const int x = 12;
{
  enum { x = x };
//             ^
//             |
//             +---// Point of declaration
//                 // compiler has to reach to "}" then
//                 // there is just one `x`, the `x` of `const int x=12`
}
```
- The enumerator `x` is initialized with the value of the constant `x`, namely `12`.
- [Source](https://stackoverflow.com/questions/15746271/point-of-declaration-in-c)

