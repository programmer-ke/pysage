## 1. What is an Iterable?

An **iterable** is any object that can provide an **iterator** for use in `for` loops, unpacking, comprehensions, and more. Every standard collection in Python is iterable.

```python
# Lists, tuples, strings, dicts, sets, files – all iterable
for item in [1, 2, 3]:
    print(item)

a, b, c = "abc"   # unpacking works because strings are iterable
```

The key idea: an iterable **produces** an iterator when needed.

---

## 2. How `iter()` Works – The Two‑Step Protocol

When Python needs to iterate over an object `x`, it calls the built‑in `iter(x)`:

1. If `x` has a method `__iter__`, it calls that to obtain an iterator.
2. If `__iter__` is missing but `x` has `__getitem__` (indexed from 0), Python creates an iterator that fetches items sequentially until `IndexError` is raised.
3. Otherwise, a `TypeError` is raised.

```python
class OldStyleIterable:
    def __getitem__(self, index):
        if index >= 3:
            raise IndexError
        return index * 10

obj = OldStyleIterable()
for val in obj:
    print(val)   # prints 0, 10, 20
```

This fallback to `__getitem__` is a form of **duck typing** – if it walks like a sequence, it can be iterated.

---

## 3. Duck Typing vs. Goose Typing

- **Duck typing**: “If it quacks like a duck, it’s a duck.” The most flexible way to check if something is iterable is to **try calling `iter()` on it** and catch `TypeError`.
- **Goose typing**: Checking for an explicit interface, e.g., `isinstance(x, abc.Iterable)`. This is stricter because it requires `__iter__` to be defined.

```python
from collections.abc import Iterable

def is_iterable_duck(obj):
    try:
        iter(obj)
        return True
    except TypeError:
        return False

# A class with only __getitem__ passes duck typing but fails goose typing
class OnlyGetItem:
    def __getitem__(self, i):
        if i >= 3: raise IndexError
        return i

print(is_iterable_duck(OnlyGetItem()))   # True
print(isinstance(OnlyGetItem(), Iterable))  # False
```

The notes recommend the duck‑typing approach for maximum flexibility.

---

## 4. Creating an Iterable from a Callable

The two‑argument form `iter(callable, sentinel)` repeatedly calls the callable until it returns the sentinel value. This is handy for reading files in chunks or processing streams.

```python
# Read blocks of 4 bytes until empty bytes returned
with open('data.bin', 'rb') as f:
    for block in iter(lambda: f.read(4), b''):
        print(block)
```

Here, `f.read(4)` is called each iteration; when it returns `b''` (the sentinel), the loop stops.

---

## 5. The Iterator Interface

An **iterator** is an object that implements two methods:

- `__next__()` – returns the next item or raises `StopIteration` when exhausted.
- `__iter__()` – returns `self`, so the iterator can be used wherever an iterable is expected.

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for num in Countdown(3):
    print(num)   # prints 3, 2, 1
```

Because `__iter__` returns `self`, you can use the same object in multiple `for` loops – but be careful: the iterator is **stateful** and will be exhausted after the first loop.

---

## 6. Checking for an Iterator

The best way to check if an object is an iterator is `isinstance(x, abc.Iterator)`. This works because `abc.Iterator` uses a `__subclasshook__` that checks for both `__iter__` and `__next__` methods.

```python
from collections.abc import Iterator

print(isinstance(Countdown(5), Iterator))   # True
print(isinstance([1,2,3], Iterator))        # False (list is iterable, not iterator)
```

This structural check is more reliable than checking for a specific class.

---

## 7. Independent Traversals

A well‑behaved iterable should support **multiple independent traversals**. Each call to `iter()` must return a **fresh** iterator with its own state.

```python
class MultiIterable:
    def __init__(self, data):
        self.data = data

    def __iter__(self):
        return iter(self.data)   # returns a new list iterator each time

mi = MultiIterable([1, 2, 3])
it1 = iter(mi)
it2 = iter(mi)
print(next(it1))   # 1
print(next(it2))   # 1  (independent!)
```

If `__iter__` returned `self`, the two iterators would share state, which is usually undesirable.

---

## 8. Generators – The Most Compact Iterators

A **generator** is a function that uses `yield` to produce values. It automatically implements the iterator protocol. The most concise way to create an iterator is to implement `__iter__` as a generator (or return a generator expression).

```python
class Squares:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        for i in range(self.n):
            yield i * i

for sq in Squares(4):
    print(sq)   # 0, 1, 4, 9
```

Generator expressions are even shorter:

```python
squares = (i*i for i in range(4))
print(list(squares))   # [0, 1, 4, 9]
```

Generators are lazy – they produce values on demand, saving memory.

---

## 9. Composable Generators

Python’s `itertools`, `functools`, and built‑ins provide a rich set of generators that can be combined like building blocks.

```python
import itertools

# Chain multiple iterables
for item in itertools.chain([1,2], (3,4), "ab"):
    print(item)   # 1 2 3 4 'a' 'b'

# Filter with a predicate
evens = (x for x in range(10) if x % 2 == 0)

# Map and reduce
from functools import reduce
sum_of_squares = reduce(lambda a,b: a+b, (x*x for x in range(4)))
print(sum_of_squares)   # 14
```

This composability lets you build complex data pipelines without writing explicit loops.

---

## 10. `yield from` – Delegating to a Sub‑Generator

`yield from` allows a generator to delegate part of its operation to another generator (or any iterable). It transparently forwards values and exceptions.

```python
def sub_generator():
    yield 'A'
    yield 'B'

def main_generator():
    yield 'Start'
    yield from sub_generator()
    yield 'End'

for val in main_generator():
    print(val)   # Start, A, B, End
```

`yield from` also handles `send()`, `throw()`, and `close()` correctly, making it essential for building coroutine‑based systems.

---

## 11. Generators vs. Coroutines

Despite similar syntax (both use `yield`), generators and coroutines serve **different purposes**:

- **Generators** produce data for iteration. They are **pull‑based**: the caller pulls values via `next()`.
- **Coroutines** consume data and may produce data not meant for iteration. They are **push‑based**: data is sent in via `send()`.

```python
# Generator – produces values
def gen():
    yield 1
    yield 2

g = gen()
print(next(g))   # 1
# g.send(10)     # TypeError: can't send non-None value to a just-started generator

# Coroutine – consumes values
def coro():
    print('Ready')
    while True:
        received = yield
        print(f'Got: {received}')

c = coro()
next(c)          # prime the coroutine (advance to first yield)
c.send('Hello')  # Got: Hello
c.send(42)       # Got: 42
c.close()        # clean shutdown
```

Calling `send()` on a generator that isn’t designed for it raises an error. Coroutines are explicitly written to accept data.

---

## 12. Coroutine Lifecycle

A coroutine body often contains an **infinite loop** waiting for data. It can be:

- **Garbage collected** when no longer referenced (the interpreter calls `close()` automatically).
- **Closed explicitly** via `.close()`, which injects a `GeneratorExit` exception at the suspension point.

```python
def infinite_coro():
    try:
        while True:
            value = yield
            print(f'Processing {value}')
    except GeneratorExit:
        print('Coroutine closed, cleaning up')

c = infinite_coro()
next(c)          # prime
c.send(1)        # Processing 1
c.close()        # Coroutine closed, cleaning up
```

If you forget to close a coroutine, it will be garbage‑collected when it goes out of scope, but explicit closing is cleaner.

---

## Summary

- **Iterables** provide iterators via `__iter__` (or `__getitem__`).
- **Iterators** implement `__next__` and `__iter__` (returning `self`).
- **Generators** are the easiest way to create iterators, using `yield`.
- **`yield from`** delegates to sub‑generators.
- **Coroutines** are like generators but designed to consume data via `send()`.
- Use `itertools` and friends to compose powerful, memory‑efficient pipelines.
