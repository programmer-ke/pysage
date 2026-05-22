# Special Methods for Sequences

Python’s data model gives you the power to make your custom objects behave like built‑in sequences—supporting iteration, slicing, indexing, and more. By implementing just a few special methods, you unlock a wealth of functionality.

---

## 1. The Minimal Sequence: `__getitem__` and `__len__`

Implementing `__getitem__` and `__len__` is enough to make your object iterable, support `in` checks, indexing, and even slicing.

```python
class MyList:
    def __init__(self, items):
        self._items = list(items)

    def __getitem__(self, index):
        return self._items[index]

    def __len__(self):
        return len(self._items)

ml = MyList([10, 20, 30, 40])

# Indexing
print(ml[0])     # 10

# Length
print(len(ml))   # 4

# Iteration
for item in ml:
    print(item)  # 10, 20, 30, 40

# Membership
print(20 in ml)  # True
```

Without any inheritance, your object suddenly behaves like a sequence.

---

## 2. Slicing Creates Slice Objects

When you write `seq[a:b:c]`, Python builds a `slice(a, b, c)` object and passes it to `__getitem__`. Your code can then inspect and use that slice.

```python
class SlicedList:
    def __init__(self, items):
        self._items = list(items)

    def __getitem__(self, idx):
        if isinstance(idx, slice):
            print(f"Got a slice: start={idx.start}, stop={idx.stop}, step={idx.step}")
            return self._items[idx]
        return self._items[idx]

sl = SlicedList([0, 1, 2, 3, 4, 5])
print(sl[1:4])    # Got a slice: start=1, stop=4, step=None → [1, 2, 3]
print(sl[::2])    # Got a slice: start=None, stop=None, step=2 → [0, 2, 4]
```

You can implement custom slicing logic by interpreting the `slice` object.

---

## 3. Safe Representations with `reprlib.repr`

For objects that may be large or contain recursive structures, `reprlib.repr` limits output length and handles recursion gracefully.

```python
import reprlib

class LargeSequence:
    def __init__(self, data):
        self.data = list(data)

    def __repr__(self):
        return f"LargeSequence({reprlib.repr(self.data)})"

ls = LargeSequence(range(100))
print(repr(ls))
# LargeSequence([0, 1, 2, 3, 4, 5, ...])   # truncated
```

For recursive structures, `reprlib.repr` avoids infinite loops:

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.parent = None

    def __repr__(self):
        return f"Node({self.value!r}, parent={reprlib.repr(self.parent)})"

root = Node('root')
child = Node('child')
child.parent = root
root.parent = child  # circular

print(repr(root))  # Node('root', parent=Node('child', parent=...))
```

---

## 4. `repr()` Should Never Raise an Exception

Because `repr()` is used for debugging, it should always return a useful string. If something goes wrong, catch the error and provide fallback information.

```python
class Robust:
    def __init__(self, data):
        self.data = data

    def __repr__(self):
        try:
            return f"Robust({self.data!r})"
        except Exception as e:
            return f"<Robust (repr failed: {e})>"

r = Robust(print)
print(repr(r))  # Robust(<built-in function print>)

r2 = Robust(object())
print(repr(r2))  # Robust(<object object at 0x...>)
```

---

## 5. Protocols: Informal Interfaces

In Python, a **protocol** is an interface defined by documentation and convention, not by code. You can implement only the parts you need.

```python
class PartiallyIterable:
    # Only __getitem__; no __iter__, no __len__
    def __init__(self, items):
        self._items = items

    def __getitem__(self, idx):
        return self._items[idx]

p = PartiallyIterable([1, 2, 3])
for x in p:
    print(x)  # Works, because Python falls back to __getitem__ for iteration
```

You don't have to implement the full `Sequence` ABC; Python's data model adapts. However, with static type checkers like mypy, you may need `typing.Protocol` to indicate intent.

---

## 6. `slice.indices(len)` Handles Missing/Out‑of‑Range Slices

`slice.indices(length)` returns a `(start, stop, stride)` tuple that is guaranteed to be valid for a sequence of the given length.

```python
class Matrix:
    def __init__(self, rows):
        self._rows = list(rows)
        self._n = len(rows[0]) if rows else 0

    def __getitem__(self, idx):
        if isinstance(idx, slice):
            start, stop, stride = idx.indices(len(self._rows))
            return [self._rows[i] for i in range(start, stop, stride)]
        return self._rows[idx]

m = Matrix([[1,2],[3,4],[5,6]])
print(m[0:10])  # start=0, stop=3, stride=1 → all rows, no IndexError
```

This eliminates off‑by‑one errors and negative index handling.

---

## 7. `operator.index` and `__index__`

`operator.index(obj)` calls `obj.__index__()` and raises a `TypeError` if the object cannot be used as an integer index. Use it to validate index types.

```python
import operator

class MyInt:
    def __init__(self, val):
        self.val = val
    def __index__(self):
        return self.val

idx = MyInt(2)
print(operator.index(idx))  # 2

# Using with sequences
lst = [10, 20, 30, 40]
print(lst[idx])  # 30

# operator.index(3.5)  # TypeError
```

Custom numeric types that represent indices should implement `__index__`.

---

## 8. Excessive `isinstance` is a Code Smell

Relying heavily on `isinstance` checks often indicates a design that could be improved with polymorphism. Instead of type‑checking, trust the object to behave correctly (duck typing).

```python
# ❌ Bad: many isinstance checks
def sum_all(items):
    if not isinstance(items, list):
        raise TypeError("Must be a list")
    return sum(items)  # but what about tuples, sets, generators?

# ✅ Better: accept any iterable
def sum_all(items):
    total = 0
    for x in items:
        total += x
    return total

print(sum_all([1,2,3]))   # 6
print(sum_all((1,2,3)))   # 6
print(sum_all({1,2,3}))   # 6 (sum of set elements)
```

---

## 9. `__getattr__` as a Fallback

`__getattr__` is called only when regular attribute lookup fails. It's the last resort, not the first.

```python
class Fallback:
    def __init__(self):
        self.existing = "I'm here"

    def __getattr__(self, name):
        print(f"'{name}' not found, providing default")
        return f"Generated_{name}"

fb = Fallback()
print(fb.existing)   # I'm here   (normal lookup)
print(fb.missing)    # 'missing' not found, providing default → Generated_missing
```

Use `__getattr__` to compute values lazily or provide virtual attributes.

---

## 10. Pair `__getattr__` with `__setattr__`

When you override how attributes are retrieved, you usually need to control how they are set to keep the object consistent.

```python
class Consistent:
    def __init__(self):
        self._data = {}

    def __getattr__(self, name):
        if name.startswith('_'):
            raise AttributeError(name)
        return self._data.get(name, f"default_{name}")

    def __setattr__(self, name, value):
        if name.startswith('_'):
            super().__setattr__(name, value)
        else:
            self._data[name] = value

c = Consistent()
c.color = "red"
print(c.color)      # red
print(c.size)       # default_size (getattr fallback)
c._internal = 5
print(c._internal)  # 5 (normal attribute)
```

Be careful: `__setattr__` is called for *every* assignment; you need a way to store real instance variables (like calling `super().__setattr__`).

---

## 11. `functools.reduce` Needs an Initializer

When using `reduce`, always provide an initializer to handle empty sequences. The initializer should be the **identity value** of the operation.

```python
from functools import reduce

numbers = []

# ❌ Without initializer: raises TypeError on empty sequence
# reduce(lambda x, y: x + y, numbers)  # TypeError

# ✅ With identity value (0 for addition)
result = reduce(lambda x, y: x + y, numbers, 0)
print(result)  # 0

# Other identity examples:
# Multiplication: identity = 1
result = reduce(lambda x, y: x * y, [2,3,4], 1)  # 24
# Bitwise OR: identity = 0
result = reduce(lambda x, y: x | y, [1,2,4], 0)  # 7
```

The initializer also determines the type of the empty‑sequence result.

---

## 12. Transposing a Matrix with `zip`

`zip` can elegantly transpose a matrix represented as nested iterables (e.g., list of lists).

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# Transpose using zip with * unpacking
transposed = list(zip(*matrix))
print(transposed)
# [(1, 4, 7), (2, 5, 8), (3, 6, 9)]

# To get lists instead of tuples:
transposed_lists = [list(row) for row in zip(*matrix)]
print(transposed_lists)
# [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

This works because `zip(*matrix)` feeds each row as an argument, and then `zip` picks the first elements, then the second, etc.

---

## Summary

| Technique | Why it matters |
|---|---|
| `__getitem__` + `__len__` | Instant sequence behavior |
| `seq[a:b:c]` → `slice` | Infinite custom slicing possibilities |
| `reprlib.repr` | Safe output for large / recursive data |
| Robust `repr()` | Never breaks debugging |
| Protocols | Flexible, partial implementation; Python adapts |
| `slice.indices()` | Error‑free handling of any slice |
| `operator.index` / `__index__` | Validate custom indices |
| Avoid `isinstance` chains | Favor duck typing and polymorphism |
| `__getattr__` fallback | Virtual / computed attributes |
| Pairing `__getattr__` with `__setattr__` | Consistent attribute behavior |
| `reduce` initializer | Safety and correctness for empty sequences |
| `zip(*matrix)` | Elegant matrix transpose |

These special methods turn your custom objects into first‑class citizens of the Python ecosystem, letting them work seamlessly with built‑in functions and language constructs.
