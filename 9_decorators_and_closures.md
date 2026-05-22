## Decorators and Closures

### 1. What is a Decorator?

A decorator is a **callable** (usually a function or a class) that takes a function and returns a replacement function. It can modify or enhance behavior.

```python
# A simple decorator
def my_decorator(func):
    def wrapper():
        print("Something before the function.")
        func()
        print("Something after the function.")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Something before the function.
# Hello!
# Something after the function.
```

**Key point:** The decorator *may replace* the decorated function with a different one. Here, `say_hello` is replaced by `wrapper`.

---

### 2. Decorators Execute at Import Time

Decorators run when the **module is loaded**, not when the decorated function is called.

```python
# decorator_demo.py
def register(func):
    print(f"Registering {func.__name__}")
    return func

@register
def f1():
    pass

@register
def f2():
    pass

print("Module loaded.")
```

```python
# When you import this module:
>>> import decorator_demo
Registering f1
Registering f2
Module loaded.
```

The decorator runs immediately at import time, even though `f1` and `f2` are never called.

---

### 3. Closures and Free Variables

A **free variable** is a variable used in a function that is not a local variable or a parameter. A **closure** is a function that retains access to free variables from its enclosing scope, even after that scope is gone.

```python
def make_multiplier(factor):
    # factor is a free variable in the inner function
    def multiplier(x):
        return x * factor
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15
```

Here, `factor` is a free variable inside `multiplier`. The closure "remembers" the value of `factor` (2 for `double`, 3 for `triple`) even after `make_multiplier` has returned.

You can inspect the closure:

```python
print(double.__closure__[0].cell_contents)  # 2
print(triple.__closure__[0].cell_contents)  # 3
```

---

### 4. Lexical Scoping (not Dynamic Scoping)

Python uses **lexical scoping**: free variables are resolved in the scope where the function was *defined*, not where it is *executed*.

```python
x = "global"

def outer():
    x = "outer"
    def inner():
        print(x)  # Which x?
    return inner

f = outer()
f()  # Prints "outer" (lexical), not "global" (dynamic)
```

With lexical scoping, `inner` looks up `x` in the scope where it was defined (`outer`), not where it is called (global scope).

---

### 5. `functools.wraps` Preserves Metadata

When a decorator replaces a function, the original function's metadata (`__name__`, `__doc__`, etc.) is lost. `functools.wraps` copies these attributes to the wrapper.

```python
import functools

def without_wraps(func):
    def wrapper(*args, **kwargs):
        """Wrapper docstring."""
        return func(*args, **kwargs)
    return wrapper

def with_wraps(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        """Wrapper docstring."""
        return func(*args, **kwargs)
    return wrapper

@without_wraps
def greet1():
    """Say hello."""
    pass

@with_wraps
def greet2():
    """Say hello."""
    pass

print(greet1.__name__)  # wrapper
print(greet1.__doc__)   # Wrapper docstring.

print(greet2.__name__)  # greet2
print(greet2.__doc__)   # Say hello.
```

Always use `@functools.wraps` when writing decorators to preserve the original function's identity.

---

### 6. `functools.lru_cache` for Memory-Constrained Caching

`functools.cache` is a simple unbounded cache. When memory is a concern, use `functools.lru_cache` with a `maxsize` limit.

```python
import functools

# Unbounded cache - could grow indefinitely
@functools.cache
def expensive_computation_unbounded(n):
    print(f"Computing {n}...")
    return n * n

# LRU cache with a size limit - safer for memory
@functools.lru_cache(maxsize=128)
def expensive_computation_bounded(n):
    print(f"Computing {n}...")
    return n * n

print(expensive_computation_bounded(10))  # Computing 10... \n 100
print(expensive_computation_bounded(10))  # 100 (cached, no recomputation)
```

`lru_cache` evicts the least recently used entries when the cache reaches `maxsize`, keeping memory usage bounded.

---

### 7. `@singledispatch` for Generic Functions

`@singledispatch` creates a **generic function**—a function with multiple implementations depending on the type of the first argument.

```python
from functools import singledispatch
import numbers

@singledispatch
def describe(obj):
    """Default implementation."""
    return f"Unknown type: {type(obj).__name__}"

@describe.register
def _(obj: str):
    return f"String of length {len(obj)}"

@describe.register
def _(obj: numbers.Integral):
    return f"Integer: {obj}"

@describe.register
def _(obj: list):
    return f"List with {len(obj)} items"

print(describe("hello"))   # String of length 5
print(describe(42))        # Integer: 42
print(describe([1, 2, 3])) # List with 3 items
print(describe(3.14))      # Unknown type: float
```

**Best practice:** Register implementations for **abstract types** (like `numbers.Integral`) rather than concrete types (like `int`). This makes the generic function work with a wider variety of types (e.g., `int`, `numpy.int64`, etc.).

```python
# Better: uses abstract type
@describe.register
def _(obj: numbers.Integral):
    return f"Integer: {obj}"

# Worse: only works with int
@describe.register
def _(obj: int):
    return f"Integer: {obj}"
```

---

### 8. `singledispatch` is for Modular Extension, not Method Overloading

Using `singledispatch` to overload methods *within the same class* concentrates too much responsibility. Its best use is supporting **modular extension** for yet-unforeseen cases.

```python
# Good: modular extension from outside
# In a library:
@singledispatch
def render(obj):
    raise NotImplementedError(f"No renderer for {type(obj)}")

# In user code (separate module):
class MyCustomType:
    def __init__(self, data):
        self.data = data

@render.register
def _(obj: MyCustomType):
    return f"Rendered: {obj.data}"

obj = MyCustomType("hello")
print(render(obj))  # Rendered: hello
```

This allows users to extend the behavior without modifying the original code.

---

### 9. Decorators as Classes with `__call__`

For decorators that need to maintain state, implement them as classes with a `__call__` method.

```python
import functools

class CountCalls:
    """Decorator that counts how many times a function is called."""
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
greet("Bob")
greet("Charlie")
# Output:
# Call 1 of greet
# Hello, Alice!
# Call 2 of greet
# Hello, Bob!
# Call 3 of greet
# Hello, Charlie!
```

Using a class with `__call__` is the recommended way to implement decorators that need to persist state across calls.

---

### Summary

| Concept | Key Idea |
|---|---|
| Decorator | Callable that replaces/enhances a function |
| Import time execution | Decorators run when the module loads |
| Free variable | Variable from enclosing scope, not local |
| Closure | Function that retains access to free variables |
| Lexical scoping | Variables resolved at definition, not execution |
| `functools.wraps` | Preserves `__name__`, `__doc__` of wrapped function |
| `functools.lru_cache` | Bounded cache for memory-constrained scenarios |
| `@singledispatch` | Generic function with type-based dispatch |
| Abstract types | Prefer `numbers.Integral` over `int` for dispatch |
| Modular extension | `singledispatch` best for external extensibility |
| Class-based decorators | Use `__call__` for stateful decorators |
