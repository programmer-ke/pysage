# Interfaces, Protocols, and ABCs

Python's approach to interfaces is uniquely flexible. You have three main tools: **dynamic protocols** (duck typing), **static protocols** (`typing.Protocol`), and **Abstract Base Classes** (`collections.abc`). Let's explore each and understand when to use them.

---

## 1. Protocols as Informal (Dynamic) Interfaces

In Python, a **protocol** is an informal interface—defined by documentation and convention, not by code. You don't need to inherit from anything; you just implement the methods the protocol expects.

```python
# The "iterable protocol" requires only __iter__
class Countdown:
    """An iterable that counts down from n to 1."""
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        self.current = self.n
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# No inheritance. No registration. It just works.
for num in Countdown(5):
    print(num, end=' ')  # 5 4 3 2 1
```

Python calls these **dynamic interfaces**—the interpreter itself understands them (like the iterator protocol) without any formal declaration.

---

## 2. `typing.Protocol` for Static Interfaces

`typing.Protocol` allows you to formally define an interface that type checkers can verify at development time—without requiring runtime inheritance.

```python
from typing import Protocol

class Drawable(Protocol):
    """Anything that can be drawn has a draw() method."""
    def draw(self) -> str:
        ...

class Circle:
    def draw(self) -> str:
        return "○"

class Square:
    def draw(self) -> str:
        return "□"

class Dog:
    def bark(self) -> str:
        return "Woof!"

def render(shape: Drawable) -> None:
    print(f"Rendering: {shape.draw()}")

render(Circle())   # Works: Rendering: ○
render(Square())   # Works: Rendering: □
# render(Dog())    # Type checker rejects this: Dog has no draw()
```

At *runtime*, nothing is enforced. But mypy/pyright will catch violations, giving you **static duck typing**.

---

## 3. The Python Data Model Philosophy

The interpreter is designed to cooperate with dynamic protocols. It compensates for missing dunder methods.

```python
class PartialSequence:
    """Only implements __getitem__—no __iter__, no __len__."""
    def __init__(self, items):
        self._items = list(items)

    def __getitem__(self, index):
        return self._items[index]

ps = PartialSequence([10, 20, 30])

# Python falls back to __getitem__ for iteration
for item in ps:
    print(item)  # Works!

# Falls back to __getitem__ for containment
print(20 in ps)  # True

# Falls back to __getitem__ for reversed
print(list(reversed(ps)))  # [30, 20, 10]
```

The interpreter fills in the gaps. This is the essence of Python's pragmatic design.

---

## 4. `collections.abc` Makes Protocols Explicit

The `collections.abc` module provides formal ABCs for common protocols: `Sequence`, `Mapping`, `Set`, `Iterator`, etc.

```python
from collections.abc import Sequence, MutableSequence

# Built-in types are recognized
print(isinstance([1, 2, 3], Sequence))        # True
print(isinstance((1, 2, 3), Sequence))        # True
print(isinstance({1, 2, 3}, Sequence))        # False
print(isinstance("hello", Sequence))          # True

# Custom types can register or inherit
class MyList(Sequence):
    def __init__(self, *items):
        self._items = list(items)

    def __getitem__(self, idx):
        return self._items[idx]

    def __len__(self):
        return len(self._items)

    # Sequence provides __contains__, __iter__, __reversed__, index, count
    # for free because we implemented __getitem__ and __len__!
```

By inheriting from `Sequence`, you get `__contains__`, `__iter__`, `__reversed__`, `index()`, and `count()` for free.

---

## 5. Following Established Protocols Improves Reusability

When you implement standard protocols, your objects work with the entire Python ecosystem.

```python
from collections.abc import Mapping

class LazyDict(Mapping):
    """A read-only mapping that computes values on demand."""
    def __init__(self, compute_fn, keys):
        self._compute = compute_fn
        self._keys = set(keys)

    def __getitem__(self, key):
        if key not in self._keys:
            raise KeyError(key)
        return self._compute(key)

    def __len__(self):
        return len(self._keys)

    def __iter__(self):
        return iter(self._keys)

ld = LazyDict(lambda k: k.upper(), ['a', 'b', 'c'])

# Works with all mapping-aware code
print(dict(ld))                    # {'a': 'A', 'b': 'B', 'c': 'C'}
print('a' in ld)                   # True
print(list(ld.keys()))             # ['a', 'b', 'c']
print({**ld})                      # {'a': 'A', 'b': 'B', 'c': 'C'}
```

---

## 6. Monkey Patching to Adapt External Code

You can dynamically add methods to classes you don't control, making them conform to a protocol—at the cost of tight coupling.

```python
from collections.abc import Sequence

# Third-party class we can't modify
class ThirdPartyList:
    def __init__(self, items):
        self.items = items

# Monkey patch to make it a Sequence
def _getitem(self, idx):
    return self.items[idx]

def _len(self):
    return len(self.items)

ThirdPartyList.__getitem__ = _getitem
ThirdPartyList.__len__ = _len

# Also register it as a virtual subclass of Sequence
Sequence.register(ThirdPartyList)

tpl = ThirdPartyList([1, 2, 3])
print(isinstance(tpl, Sequence))   # True
for x in tpl:
    print(x)                       # Works!
```

Use sparingly—monkey patching creates tight coupling and can break if the patched library changes.

---

## 7. Duck Typing for Defensive Programming

Call `list()` or `iter()` on arguments early to validate them and get clear error messages.

```python
def process_items(items):
    # Convert to list eagerly—fails fast if not iterable
    items = list(items)
    if not items:
        raise ValueError("Must have at least one item")
    return sum(items) / len(items)

# Clear error at the call site
# process_items(42)  # TypeError: 'int' object is not iterable

print(process_items([1, 2, 3]))  # 2.0
print(process_items((4, 5, 6)))  # 5.0
print(process_items(x for x in range(10, 13)))  # 11.0
```

This "failing early" pattern catches errors at the point of entry, not deep inside your logic.

---

## 8. Duck Typing is More Robust and Expressive Than Static Typing

Dynamic protocols allow patterns that are hard to express in static type systems.

```python
def flatten(nested):
    """Recursively flatten nested iterables."""
    for item in nested:
        try:
            yield from flatten(item)
        except TypeError:
            yield item

# Works with any mix of nesting
data = [1, [2, [3, 4], 5], 6]
print(list(flatten(data)))  # [1, 2, 3, 4, 5, 6]

# Complex dynamic behavior
mixed = [[1, 2], (3, [4, 5]), {6, 7}]
print(list(flatten(mixed)))  # [1, 2, 3, 4, 5, 6, 7]
```

This kind of deeply nested, heterogeneous processing is hard to type statically but natural with duck typing.

---

## 9. Goose Typing: ABCs + `isinstance`

**Goose typing** means inheriting from ABCs and using `isinstance`/`issubclass` to verify interface conformance.

```python
from collections.abc import MutableSequence

class ValidatedList(MutableSequence):
    def __init__(self, validator, *items):
        self._validator = validator
        self._items = [self._validator(item) for item in items]

    def __getitem__(self, idx):
        return self._items[idx]

    def __setitem__(self, idx, value):
        self._items[idx] = self._validator(value)

    def __delitem__(self, idx):
        del self._items[idx]

    def __len__(self):
        return len(self._items)

    def insert(self, idx, value):
        self._items.insert(idx, self._validator(value))

def uses_sequence(seq: MutableSequence):
    print(f"Processing {len(seq)} items")
    seq[0] = "updated"
    return seq

vl = ValidatedList(str, "a", "b", "c")
print(isinstance(vl, MutableSequence))  # True
print(uses_sequence(vl))                # Works
```

**Rule:** `isinstance` with ABCs is acceptable. With concrete classes, it limits polymorphism.

---

## 10. `isinstance` Chains are a Code Smell

```python
# ❌ Code smell: chain of isinstance
def describe(obj):
    if isinstance(obj, int):
        return f"Integer: {obj}"
    elif isinstance(obj, str):
        return f"String: {obj}"
    elif isinstance(obj, list):
        return f"List: {obj}"
    else:
        return f"Unknown: {type(obj).__name__}"

# ✅ Better: single dispatch or polymorphism
from functools import singledispatch

@singledispatch
def describe(obj):
    return f"Unknown: {type(obj).__name__}"

@describe.register
def _(obj: int):
    return f"Integer: {obj}"

@describe.register
def _(obj: str):
    return f"String: {obj}"

@describe.register
def _(obj: list):
    return f"List: {len(obj)} items"

print(describe(42))         # Integer: 42
print(describe("hello"))    # String: hello
```

Chains of `isinstance` mean you're fighting polymorphism. Use `singledispatch` or proper OO design instead.

---

## 11. ABCs are for Frameworks and Libraries

Most application developers don't need to create custom ABCs. They're primarily a tool for framework authors.

```python
# Library author creates an ABC
from abc import ABC, abstractmethod

class Plugin(ABC):
    @abstractmethod
    def initialize(self, config):
        ...

    @abstractmethod
    def execute(self, data):
        ...

    def description(self):
        """Concrete method: subclasses can override or use this default."""
        return f"Plugin: {self.__class__.__name__}"
```

---

## 12. Concrete Methods in ABCs Use Only the Public Interface

Concrete methods in ABCs must work for *any* subclass, so they only call other methods in the ABC's interface—they have no knowledge of instance internals.

```python
from abc import ABC, abstractmethod

class SizedContainer(ABC):
    @abstractmethod
    def __len__(self):
        ...

    def is_empty(self):
        """Concrete method: relies only on __len__, which is in the interface."""
        return len(self) == 0

    def is_large(self, threshold=100):
        """Another concrete method, also based purely on __len__."""
        return len(self) > threshold

class MyList(SizedContainer):
    def __init__(self, items):
        self._items = list(items)
    def __len__(self):
        return len(self._items)

ml = MyList([1, 2, 3])
print(ml.is_empty())    # False
print(ml.is_large(2))   # True
```

The concrete methods work because they only depend on the abstract `__len__`.

---

## 13. Abstract Methods Can Have Default Implementations

An `@abstractmethod` can include a body. Subclasses must still override it, but can call the default via `super()`.

```python
from abc import ABC, abstractmethod

class Validator(ABC):
    @abstractmethod
    def validate(self, value):
        """Default: accept anything. Override to add rules."""
        return True

class PositiveValidator(Validator):
    def validate(self, value):
        if not super().validate(value):  # Call default first
            return False
        return value > 0

class EmailValidator(Validator):
    def validate(self, value):
        if not super().validate(value):
            return False
        return '@' in value and '.' in value

pv = PositiveValidator()
print(pv.validate(5))    # True
print(pv.validate(-1))   # False
```

---

## 14. `@abstractmethod` Must Be the Innermost Decorator

When combining `@abstractmethod` with other decorators like `@property` or `@classmethod`, `@abstractmethod` must be the **innermost** one.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @property
    @abstractmethod      # innermost
    def area(self):
        ...

    @classmethod
    @abstractmethod      # innermost
    def from_dict(cls, data):
        ...

class Circle(Shape):
    def __init__(self, radius):
        self._radius = radius

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

    @classmethod
    def from_dict(cls, data):
        return cls(data['radius'])

c = Circle(5)
print(f"{c.area:.2f}")  # 78.54
```

---

## 15. Virtual Subclass Registration

You can register any class as a "virtual subclass" of an ABC—no inheritance required. Python does *not* verify that the class actually implements the interface.

```python
from collections.abc import Sequence

class MySequenceLike:
    def __init__(self, items):
        self._items = list(items)
    def __getitem__(self, idx):
        return self._items[idx]
    def __len__(self):
        return len(self._items)

# Register without checking—it's on you to get it right
Sequence.register(MySequenceLike)

msl = MySequenceLike([1, 2, 3])
print(isinstance(msl, Sequence))  # True
print(issubclass(MySequenceLike, Sequence))  # True
```

This is how built‑ins like `list`, `tuple`, and `str` are registered as virtual subclasses of `Sequence` in `collections.abc`.

---

## 16. `__subclasshook__` for Structural Typing

ABCs can implement `__subclasshook__` to dynamically inspect a class and decide whether it qualifies, enabling automatic structural subtype detection.

```python
from abc import ABC

class Duck(ABC):
    @classmethod
    def __subclasshook__(cls, C):
        if cls is Duck:
            # Check if C has both quack and walk methods
            if any("quack" in B.__dict__ for B in C.__mro__) and \
               any("walk" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented

class Mallard:
    def quack(self):
        return "Quack!"
    def walk(self):
        return "Waddle"

class Dog:
    def bark(self):
        return "Woof!"

print(isinstance(Mallard(), Duck))  # True — has quack and walk
print(isinstance(Dog(), Duck))      # False — no quack
print(issubclass(Mallard, Duck))    # True
```

---

## 17. `@runtime_checkable` for Dynamic Protocol Checking

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Quackable(Protocol):
    def quack(self) -> str:
        ...

class Duck:
    def quack(self) -> str:
        return "Quack!"

class Person:
    def quack(self) -> str:
        return "I'm pretending!"

d = Duck()
p = Person()

print(isinstance(d, Quackable))  # True
print(isinstance(p, Quackable))  # True (has quack method)
print(isinstance(42, Quackable)) # False

# Limitation: type hints are ignored at runtime
class FakeDuck:
    def quack(self) -> int:  # Returns int, not str!
        return 42

fd = FakeDuck()
print(isinstance(fd, Quackable))  # True at runtime, but mypy would catch the type mismatch
```

`@runtime_checkable` only checks for the *presence* of methods, not their type signatures. Static type checkers catch signature mismatches.

---

## 18. Single-Method Protocols Compose into Larger Protocols

```python
from typing import Protocol

class SupportsRead(Protocol):
    def read(self, size: int = ...) -> str:
        ...

class SupportsWrite(Protocol):
    def write(self, data: str) -> None:
        ...

class SupportsClose(Protocol):
    def close(self) -> None:
        ...

# Compose simpler protocols into a full FileIO protocol
class FileIO(SupportsRead, SupportsWrite, SupportsClose, Protocol):
    """A complete file-like object."""
    ...

# Any class implementing read, write, close satisfies FileIO
class MyFile:
    def read(self, size=-1):
        return "data"
    def write(self, data):
        print(f"Writing: {data}")
    def close(self):
        print("Closed")

def process_file(f: FileIO) -> None:
    data = f.read()
    f.write(data.upper())
    f.close()

process_file(MyFile())  # Type checker is happy!
```

Each protocol subclass must include `Protocol` in its direct base classes to define a *new* protocol (not just implement one).

---

## 19. Naming Conventions for Protocols

```python
from typing import Protocol

# Clear concepts → plain nouns
class Container(Protocol):
    def __contains__(self, item) -> bool: ...

class Iterator(Protocol):
    def __next__(self): ...

# Callable methods → "SupportsX"
class SupportsRead(Protocol):
    def read(self, size: int = ...) -> bytes: ...

class SupportsInt(Protocol):
    def __int__(self) -> int: ...

class SupportsRound(Protocol):
    def __round__(self, ndigits: int = ...) -> int: ...

# Attributes → "HasX"
class HasName(Protocol):
    name: str

class HasItems(Protocol):
    items: list

class HasFileNo(Protocol):
    fileno: int
```

These conventions make protocol intent immediately clear.

---

## 20. Number ABCs: Dynamic but Not Static

The `numbers` module provides a tower of numeric ABCs useful for *runtime* type checking, but static type checkers treat `int`, `float`, and `complex` as special cases.

```python
import numbers

def double_it(x):
    if isinstance(x, numbers.Real):
        return x * 2
    elif isinstance(x, numbers.Complex):
        return x * 2
    else:
        raise TypeError(f"Expected a number, got {type(x)}")

print(double_it(3))          # 6
print(double_it(3.5))        # 7.0
print(double_it(1+2j))       # (2+4j)
print(double_it(3.5))        # 7.0
# double_it("hello")         # TypeError

# For static typing, just use int, float, complex
def static_double(x: complex) -> complex:  # supports int,float,complex
    return x * 2
```

The `numbers` ABCs (`Number`, `Complex`, `Real`, `Rational`, `Integral`) detect NumPy types and other numeric libraries at runtime—something static type checkers can't do.

---

## Summary

| Concept | Use Case |
|---|---|
| **Dynamic protocols** | Duck typing; implement methods, no inheritance |
| **`typing.Protocol`** | Static duck typing for type checkers |
| **`collections.abc`** | Explicit interfaces; virtual subclasses; goose typing |
| **Monkey patching** | Adapt external code to protocols (use sparingly) |
| **Goose typing** | `isinstance` against ABCs |
| **`isinstance` chains** | Avoid—use `singledispatch` or polymorphism |
| **Concrete methods in ABCs** | Implement in terms of the abstract interface |
| **Virtual subclass** | `.register()` without inheritance |
| **`__subclasshook__`** | Automatic structural subtype detection |
| **`@runtime_checkable`** | Dynamic `isinstance` checks on protocols |
| **Protocol composition** | Build larger protocols from single-method ones |
| **`numbers` ABCs** | Runtime numeric type checking |

Python gives you a spectrum of interface tools: from the loosest (duck typing) to the most formal (ABCs with `__subclasshook__`). Choose the right level of rigor for your use case.
