### 1. `typing.overload` – describing different input combinations

Use `@overload` to tell the type checker that a function can accept several distinct signatures, and that the return type depends on the input types.

```python
from typing import overload

@overload
def handle(value: int) -> str: ...
@overload
def handle(value: str) -> bytes: ...

def handle(value):
    if isinstance(value, int):
        return f"int: {value}"      # str
    else:
        return value.encode()        # bytes
```

A static checker knows that `handle(5)` returns `str` and `handle("hello")` returns `bytes`, even though the runtime implementation has no annotations.

---

### 2. Limited expressiveness of annotations vs. Python code

Type annotations can only express a fraction of what Python can do. The built-in `max()` is a classic example. Its type stubs are huge – but a working Python implementation is just a few lines longer.

```python
# Simplified version of max – the runtime knows how to compare anything,
# but the type system needs many overloads to cover all cases.
def my_max(first, *rest, key=None, default=...):
    if not rest:
        if default is ...:
            raise TypeError("need at least one argument")
        return first
    # ... actual comparison logic
```

The actual `max` stub lists dozens of overloads to handle different combinations. The runtime code is far simpler.

---

### 3. `TypedDict` – purely a type-checker construct

`TypedDict` lets you describe expected keys and value types, but it has **zero runtime effect** – even a `json.loads` call won’t enforce it.

```python
from typing import TypedDict

class Person(TypedDict):
    name: str
    age: int

data = '{"name": "Alice", "age": "thirty"}'
person: Person = json.loads(data)   # No runtime error!
print(person["age"] + 1)            # Runtime crash here
```

You still need your own validation if the data comes from outside.

---

### 4. `typing.cast` – helping the type checker

Use `cast()` when you know more about a type than the checker can infer. It has no runtime overhead.

```python
from typing import cast

obj: object = "a string"
# Type checker sees 'object' – no upper()
# We know it's str:
s = cast(str, obj)
print(s.upper())   # OK for type checker
```

You’re telling the checker “trust me, this is a `str`”. Use sparingly.

---

### 5. Where type hints live: `__annotations__`

At import time, Python reads hints and stores them in the `__annotations__` attribute. You can retrieve them programmatically.

```python
def greet(name: str) -> str:
    return f"Hello {name}"

print(greet.__annotations__)
# {'name': <class 'str'>, 'return': <class 'str'>}

import typing
print(typing.get_type_hints(greet))   # Same, but resolves string annotations
```

Prefer `typing.get_type_hints()` because it handles forward references and string-based hints.

---

### 6. Key definitions – generic types and parameters

```python
from typing import TypeVar, Generic

T = TypeVar('T')          # Formal type parameter

class Box(Generic[T]):    # Generic type
    content: T

int_box: Box[int]         # Parameterized type (actual type parameter = int)
str_box: Box[str]         # Another parameterized type
```

- **Generic type** – `Box`
- **Formal type parameter** – `T`
- **Parameterized type** – `Box[int]`, `Box[str]`
- **Actual type parameter** – `int`, `str`

---

### 7. Invariance – mutable collections

If a generic type is **invariant**, no subtyping relation exists between `G[A]` and `G[B]` even when `B` is a subtype of `A`.

```python
class MutableBox(Generic[T]):
    def __init__(self, val: T) -> None:
        self.val = val
    def set(self, val: T) -> None:
        self.val = val
    def get(self) -> T:
        return self.val

# int is consistent-with float, but MutableBox[int] is NOT a subtype of MutableBox[float]
def set_float(b: MutableBox[float]) -> None:
    b.set(3.14)

ibox: MutableBox[int] = MutableBox(0)
# set_float(ibox)   # Type error for a type checker – would allow float into int box
```

Standard `list` is invariant for exactly this reason.

---

### 8. Covariance – immutable / read-only containers

**Covariant** means `G[B]` is a subtype of `G[A]` when `B` is a subtype of `A`. This works when data only comes **out** of the object.

```python
from typing import Sequence

def sum_seq(seq: Sequence[float]) -> float:
    return sum(seq)

ints: Sequence[int] = [1, 2, 3]
# Sequence[int] is a subtype of Sequence[float]
print(sum_seq(ints))   # OK
```

`Sequence` is read-only, so it’s safe to treat a `Sequence[int]` as a `Sequence[float]`.

---

### 9. Contravariance – “write-only” / consumers

**Contravariant** reverses the subtype relationship. For example, a callable’s parameter type is contravariant: `Callable[[int], None]` is a **supertype** of `Callable[[float], None]`.

```python
from typing import Callable

def use_processor(proc: Callable[[int], None]) -> None:
    proc(5)

def float_processor(x: float) -> None:
    print(f"Processing float: {x}")

# Callable[[float], None] is a subtype of Callable[[int], None]
use_processor(float_processor)   # OK: int → float is safe
```

If you need a function that accepts `int`, a function that accepts `float` works because `float` also accepts `int`.

---

### 10. Rules of thumb

| Direction of data | Recommended variance |
|-------------------|-----------------------|
| Out only (getter, iterator) | **Covariant** |
| In only (setter, consumer)  | **Contravariant** |
| In **and** out (mutable collection) | **Invariant** |
| When in doubt | **Invariant** (safe choice) |

Example with a custom interface:

```python
from typing import TypeVar, Generic

T_co = TypeVar('T_co', covariant=True)    # data out
T_contra = TypeVar('T_contra', contravariant=True)  # data in

class Producer(Generic[T_co]):
    def get(self) -> T_co: ...

class Consumer(Generic[T_contra]):
    def send(self, value: T_contra) -> None: ...
```

If a type parameter is used both as input and output, keep it **invariant** (omit `covariant`/`contravariant` flags).
