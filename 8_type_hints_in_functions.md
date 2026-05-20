# Type Hints in Functions

## Import Style

```python
# Keep signatures short by importing directly
from typing import TypeVar, Protocol, Callable, Iterable, Sequence, NoReturn
```

---

## PEP 483: What Is a Type?

```python
# A type = a set of values + a set of functions you can apply to them
# int: values like 1, 2, 3; operations like +, -, *, //
x: int = 5
y: str = "hello"    # str: values like "a", "ab"; operations like .upper(), .split()
```

---

## More General Types Have Narrower Interfaces

```python
# General → narrower interface
# object < abc.Sequence < abc.MutableSequence
# object has fewest methods; MutableSequence has the most
from collections.abc import Sequence, MutableSequence

def process_object(obj: object): ...       # only .__str__, .__repr__, etc.
def process_seq(seq: Sequence[int]): ...   # also .__getitem__, .__len__, ...
def process_mut_seq(seq: MutableSequence[int]): ...  # also .append, .pop, ...
```

---

## `Any` – Top and Bottom of the Type Hierarchy

```python
from typing import Any

# Any is both the most general and most specialised type
def accept_any(x: Any) -> Any:
    return x

accept_any(42)         # works
accept_any("hello")    # works
accept_any(None)       # works
```

---

## Consistent‑With Rules

```python
# 1. Subtype T2 is consistent‑with T1 (Liskov substitution)
class Animal: ...
class Dog(Animal): ...

def pet(animal: Animal) -> None: ...
pet(Dog())   # Dog is consistent‑with Animal ✅

# 2. Every type is consistent‑with Any
def takes_any(x: Any): ...
takes_any(42)
takes_any(Dog())

# 3. Any is consistent‑with every type
a: Any = "hello"
def expects_str(s: str): ...
expects_str(a)           # Any is consistent‑with str ✅
```

```python
# Among classes, consistent‑with equals subtype‑of
class Parent: ...
class Child(Parent): ...

def use_parent(p: Parent): ...
use_parent(Child())   # Child is consistent‑with Parent ✅
```

---

## Numeric Type Consistency

```python
# No class hierarchy between int, float, complex — all subclass object only
# But int is consistent‑with float, and both are consistent‑with complex

def takes_float(x: float) -> float:
    return x * 2.0

takes_float(3)       # int is consistent‑with float ✅
takes_float(3.14)    # float ✅

def takes_complex(x: complex) -> complex:
    return x * 2j

takes_complex(3)     # int consistent‑with complex ✅
takes_complex(3.14)  # float consistent‑with complex ✅
```

---

## Union Types with `|`

```python
# | operator replaces Optional[X] with X | None (Python 3.10+)
def greet(name: str | None = None) -> str:
    if name is None:
        return "Hello, stranger!"
    return f"Hello, {name}!"

# Also works with isinstance at runtime
print(isinstance("hello", str | int))  # True
print(isinstance(42, str | int))       # True
```

```python
from typing import Union

# Union is useful for types that are not consistent‑with each other
def process(data: str | bytes) -> int: ...
def process(data: Union[str, bytes]) -> int: ...  # equivalent

# Union[int, float] is redundant — int is consistent‑with float
# Just use float instead
def redundant(x: Union[int, float]) -> float:   # ❌ unnecessary
    ...
def better(x: float) -> float:                  # ✅ cleaner
    ...
```

---

## NamedTuple Consistency

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: int
    y: int

# NamedTuple is consistent‑with tuple
def use_tuple(t: tuple[int, int]) -> None:
    print(t)

p = Point(3, 4)
use_tuple(p)    # ✅ Point is consistent‑with tuple[int, int]

# Reverse is NOT true
def use_point(p: Point) -> None:
    print(p.x, p.y)

t = (3, 4)
# use_point(t)  ❌ tuple is not consistent‑with Point
```

---

## Postel's Law: Abstract in, Concrete out

```python
from collections.abc import Mapping, MutableMapping, Sequence, Iterable

# Abstract types for parameters — accept wide variety
def lookup(table: Mapping[str, int], key: str) -> int:
    return table.get(key, 0)

# Works with dict, OrderedDict, ChainMap, UserDict, etc.
lookup({'a': 1}, 'a')
lookup(OrderedDict({'b': 2}), 'b')

# Concrete types for return values — be specific
def build_table() -> dict[str, int]:
    return {'x': 10, 'y': 20}   # caller knows exactly what they get
```

```python
# Sequences: prefer Iterable for parameters, list for return
def collect(values: Iterable[int]) -> list[int]:
    return list(values)

# Accepts lists, tuples, sets, generators
collect([1, 2, 3])
collect(x * 2 for x in [1, 2, 3])  # generator works — saves memory

# Use Sequence when you need len
def average(values: Sequence[int]) -> float:
    return sum(values) / len(values)
```

---

## Numerical Tower ABCs

```python
from numbers import Integral, Real, Complex

# Useful for runtime type checking
def double(x: Real) -> Real:
    if not isinstance(x, Real):
        raise TypeError("Expected a numeric type")
    return x * 2

double(3)       # OK at runtime
double(3.14)    # OK

# Static type checkers treat int, float, complex as special cases
# — they don't use the numbers module ABCs
```

---

## TypeAlias

```python
from typing import TypeAlias

# Preferred annotation for custom type assignments
Vector: TypeAlias = list[float]
UserID: TypeAlias = int | str

def scale(v: Vector, factor: float) -> Vector:
    return [x * factor for x in v]

def get_user(uid: UserID) -> str:
    return f"User {uid}"

print(scale([1.0, 2.0], 2.0))   # [2.0, 4.0]
print(get_user(42))              # User 42
print(get_user("abc"))           # User abc
```

---

## NoReturn

```python
from typing import NoReturn

def exit_program() -> NoReturn:
    import sys
    sys.exit(1)
    # This function never returns normally

def raise_error() -> NoReturn:
    raise RuntimeError("Fatal error")
```

---

## Iterable vs Sequence for Parameters

```python
from typing import Iterable, Sequence

# Use Iterable when you don't need len — accepts generators
def strip_blanks(lines: Iterable[str]) -> list[str]:
    return [line.strip() for line in lines]

# Generator: memory‑efficient for large inputs
gen = (f"  line {i}  " for i in range(1_000_000))
result = strip_blanks(gen)

# Use Sequence when len is required
def middle(items: Sequence[int]) -> int:
    return items[len(items) // 2]

middle([1, 2, 3])   # OK
# middle(iter([1,2,3]))  ❌ iterator has no len
```

---

## TypeVar – Generic Type Variables

```python
from typing import TypeVar, Sequence

T = TypeVar('T')

def first(seq: Sequence[T]) -> T:
    return seq[0]

print(first([1, 2, 3]))            # 1, type: int
print(first(['a', 'b', 'c']))      # 'a', type: str
```

### Restricted TypeVar

```python
# Restrict to specific types
Number = TypeVar('Number', int, float)

def add(a: Number, b: Number) -> Number:
    return a + b

add(3, 4)       # int ✅
add(3.0, 4.0)   # float ✅
# add("a", "b") # ❌ str not in restriction
```

### Bounded TypeVar

```python
# Bound to a type: all consistent‑with that type work
from numbers import Real
RealNumber = TypeVar('RealNumber', bound=Real)

def multiply(a: RealNumber, b: RealNumber) -> RealNumber:
    return a * b

multiply(3, 4)        # int consistent‑with Real ✅
multiply(3.14, 2.0)   # float consistent‑with Real ✅
# multiply(3+4j, 2j)   # ❌ complex is not Real
```

---

## Protocols: Static Duck Typing

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

# No inheritance needed — any class with .draw() matches
class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())  # ✅ static duck typing
render(Square())  # ✅
```

---

## `Callable` and Variance

```python
from typing import Callable

# For optional/keyword args, replace parameter list with ...
Handler: Callable[..., None] = lambda *args, **kwargs: print(args)

# Return type is covariant: int return is subtype of float return
fn1: Callable[[], int] = lambda: 42
fn2: Callable[[], float] = fn1   # ✅ Callable[[], int] is subtype of Callable[[], float]
```


```python
# Parameter types are contravariant
fn_a: Callable[[int], None] = lambda x: None
fn_b: Callable[[float], None]

# fn_b = fn_a  ❌ not safe: the set of operations by fn_a is a superset of that of fn_b
fn_a = fn_b    # ✅ safe: anything you can do with float you can do with int
```

```python
# Practical example
from typing import Callable

def apply(f: Callable[[float], str], value: float) -> str:
    return f(value)

def int_handler(x: int) -> str:
    return str(x)

def float_handler(x: float) -> str:
    return str(x)

# apply(int_handler, 3.14)  ❌ int_handler can't handle float
apply(float_handler, 3.14)  # ✅
```

---

## Arbitrary Positional and Keyword Arguments

```python
from typing import Sequence

# *args: type specifies the type of each individual argument
def collect(*args: int) -> Sequence[int]:
    return args

collect(1, 2, 3)     # returns (1, 2, 3)

# **kwargs: type specifies the type of each value
def settings(**kwargs: str | int) -> dict[str, str | int]:
    return kwargs

settings(host="localhost", port=8080)
```

---

## Function Introspection

```python
def greet(name: str, greeting: str = "Hello") -> str:
    """Return a personalised greeting."""
    return f"{greeting}, {name}!"

# Useful attributes
print(greet.__doc__)         # Return a personalised greeting.
print(greet.__annotations__) # {'name': <class 'str'>, ..., 'return': <class 'str'>}
print(greet.__code__)        # <code object greet ...>
print(greet.__dict__)        # {} — stores arbitrary attributes
```
Inspect allows access to function attributes in a more useful way

```python
import inspect

sig = inspect.signature(greet)
print(sig)
# (name: str, greeting: str = 'Hello') -> str

for name, param in sig.parameters.items():
    print(f"{name}: default={param.default}, kind={param.kind}")
# name: default=<class 'inspect._empty'>, kind=POSITIONAL_OR_KEYWORD
# greeting: default=Hello, kind=POSITIONAL_OR_KEYWORD
```
