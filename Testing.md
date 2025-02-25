### For what?
- Find defects
- Check for right working
- Convince customer
- For yourself

### Testing ideology
1. Early tests save money
2. Excess testing does't exist
	- Избыточного тестирования не существует
	- Testing Sum(a, b): tested on (5, 4). (1, 2), (100, 500) is cringe
3. Testing shows presence of defects, not their absence
4. "80/20" rule
	- 20% of code contains 80% of errors
5. Old tests can't help in finding new mistakes
6. Testing depends on context
7. Absence of mistakes is delusion

### Testings
1. Unit tests
	- For components
2. Integration tests
	- For checking component integration
3. System tests
	- Checking integration of all components (system test)
4. Acceptance testing
	- Difference with system tests: different aspects

### Types of testings
- Loading testing
	- Checkout system overloading
- Stress testing
	- How long system can stay alive
- Scalability tests
	- How system works with scaling
	- Vertical scaling: improve system
	- Horizontal scaling: buy more system units
- "White box" testing
	- We know about system's structure
	- There is also "black box" testing
		- We don't know about system's structure
	1. Execution tests
	2. Mutation tests
		- Testing of tests
	1. Static tests
		- Ex: code review, linters: clang_tidy, clang_format
- Code coverage
	- `gcov` & `lcov`
	1. Statement coverage
		- `void f(int a, int b) { int res = a + b; if (res < 0) { cout << res; } else { cout << -res; } }`
		- Calls only `f(2, 3)`. Covered +- 70%
	2. Branch coverage
	3. Decision coverage
		- `a = -3, b = -3`
		- `if (a > 0 && b < 0)`
		- DC is 50% because of lazy calculation
	4. Path coverage
		- Checkout full path (combinations of branches)
		- exp(if_count)


### Google tests
[Link](http://google.github.io/googletest/quickstart-cmake.html)
