- [Source 1](https://stackoverflow.com/questions/46408002/c17-evaluation-order-with-operator-overloading-functions)
- [Source 2](https://timsong-cpp.github.io/cppwp/expr.call)
- [Source 3](https://stackoverflow.com/questions/38501587/what-are-the-evaluation-order-guarantees-introduced-by-c17)
- [Source 4](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0145r3.pdf)

# Overview
- Some common cases where the evaluation order has so far been _unspecified_, are specified and valid with C++17. Some undefined behaviour is now instead unspecified
```cpp
i = 1;
f(i++, i);
```
- This example was undefined, but it is now unspecified. Specifically, what is not specified is the order in which each argument to `f` is evaluated relative to the others. `i++` might be evaluated before `i`, or vice-versa. Indeed, it might evaluate a second call in a different order, despite being under the same compiler.
- However, the evaluation of each argument is required to execute completely, with all side-effects, before the execution of any other argument. So you might get `f(1, 1)` (second argument evaluated first) or `f(1, 2)` (first argument evaluated first). But you will never get `f(2, 2)` or anything else of that nature.

- Code bellow was unspecified, but it will become compatible with operator precedence so that the first evaluation of `f` will come first in the stream
```cpp
std::cout << f() << f() << f();
```
- See also postfix expressions

- But this code bellow still has unspecified evaluation order of `g`, `h`, and `j`. Note that for `getf()(g(),h(),j())`, the rules state that `getf()` will be evaluated before `g, h, j`
```cpp
f(g(), h(), j());
```

- Main rules:
	- Assignment expressions are evaluated from right to left. This includes compound assignments
	- Operands to shift operators are evaluated from left to right

# Postfix-expressions
- Postfix expressions are evaluated from left to right. This includes functions calls and member selection expressions
	- `a.b`, `a->b`, `a->*b`, `a(b1, b2, b3)` (`b1`, `b2`, `b3` order is impl. def.), `a @= b`, `a[b]`, `a << b`, `a >> b`
- The postfix-expression is sequenced before each expression in the expression-list and any default argument.
- The initialization of a parameter, including every associated value computation and side effect, is indeterminately sequenced with respect to that of any other parameter.
```cpp
#include <iostream>

struct S {
  S foo(int a) {
    std::cout << a;
    return S{};
  }
};

int main() {
  S a;
  a.foo(1).foo(2).foo(3);  // 123
}

```

# Interleaving is prohibited in C++17
However, and this is important: Even when `b1`, `b2`, `b3` are non-trivial expressions, each of them are completely evaluated _and tied to the respective function parameter_ before the other ones are started to be evaluated.

### Problem overview
In C++14, the following was unsafe:
```cpp
void foo(std::unique_ptr<A>, std::unique_ptr<B>);

foo(std::unique_ptr<A>(new A), std::unique_ptr<B>(new B));
```
There are four operations that happen here during the function call
1. `new A`
2. `unique_ptr<A>` constructor
3. `new B`
4. `unique_ptr<B>` constructor

The ordering of these was completely unspecified, and so a perfectly valid ordering is (1), (3), (2), (4). If this ordering was selected and (3) throws, then the memory from (1) leaks - we haven't run (2) yet, which would've prevented the leak.

### Prohibit interleaving
In C++17, the new rules prohibit interleaving. From `[intro.execution]`:
> For each function invocation F, for every evaluation A that occurs within F and every evaluation B that does not occur within F but is evaluated on the same thread and as part of the same signal handler (if any), either A is sequenced before B or B is sequenced before A.
> ...
> In other words, function executions do not interleave with each other.

This leaves us with two valid orderings: (1), (2), (3), (4) or (3), (4), (1), (2). It is unspecified which ordering is taken, but both of these are safe. All the orderings where (1) (3) both happen before (2) and (4) are now prohibited.

### Example
```cpp
#include <iostream>

struct S {
  S(int b) : a(b) {}

  int a;
};

int operator<<(S a, int b) {
  std::cout << a.a << ' ' << b << '\n';
  return 0;
}

int main() {
  int i, j;
  int x = S(i = 1) << (i = 2);
  int y = operator<<(S(j = 1), j = 2);
}
```

GCC output: 12 11
- Expected: 12 12
- The same problem: [link](https://stackoverflow.com/questions/16923010/order-of-parameter-evaluation-of-function-call-in-gcc)

Clang output: 12 12 (as expected)
