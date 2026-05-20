# Sequences

## Classification by Mutability

### Mutable Sequences
```python
# Types: list, bytearray, array.array, collections.deque
from collections import deque
from collections.abc import MutableSequence

lst = [1, 2, 3]
lst[0] = 100
print(lst)  # [100, 2, 3]

dq = deque([4, 5, 6])
dq.appendleft(3)
print(dq)  # deque([3, 4, 5, 6])

# Registered as virtual subclasses of abc.MutableSequence
print(isinstance(lst, MutableSequence))  # True
print(isinstance(dq, MutableSequence))   # True
```

### Immutable Sequences
```python
# Types: tuple, str, bytes
t = (1, 2, 3)
# t[0] = 100  ❌ TypeError: 'tuple' object does not support item assignment

s = "hello"
# s[0] = 'H'  ❌ TypeError: 'str' object does not support item assignment

from collections.abc import Sequence
print(isinstance(t, Sequence))  # True
print(isinstance(s, Sequence))  # True
```

---

## Classification by Element Storage

### Container Sequences
```python
# list, tuple, deque store references to objects
# Each object has its own header and metadata
l = [[]] * 2
l[0].append(1)
print(l)  # [[1], [1]]  — both elements refer to the same list object

t = ([0], [0])
t[0].append(1)
print(t)  # ([0, 1], [0])  — references are independent
```

### Flat Sequences
```python
# str, bytes, array.array store raw values contiguously
import array
import sys

arr = array.array('f', [1.0, 2.0, 3.0])  # flat: 3 floats
tup = (1.0, 2.0, 3.0)                    # container: references to float objects

print(sys.getsizeof(arr))   # ~ 72 bytes (compact)
print(sys.getsizeof(tup))   # ~ 72 bytes for tuple object + each float object
```

---

## Tuples as Records
```python
# Use tuples as lightweight records, unpack fields instead of indexing
person = ("Alice", 30, "Engineer")
name, age, role = person          # unpacking
print(f"{name} is {age} years old and works as {role}.")
# Alice is 30 years old and works as Engineer.
```

---

## Unpacking
```python
# Avoid index‑based assignment
coords = (10, 20)
x, y = coords          # parallel assignment

# Star expressions capture leftovers
first, *rest = [1, 2, 3, 4]
print(first)  # 1
print(rest)   # [2, 3, 4]

# Unpacking supports iterators
it = iter("abc")
a, b, c = it
print(a, b, c)  # a b c
```

---

## Sequence Pattern Matching
```python
command = ["quit", "--force"]

# Match sequence of exact length
match command:
    case ["start", arg]:        # sequence of 2 items
        print(f"Starting with {arg}")
    case ["quit", *flags]:      # sequence of 2+ items; capture rest
        print(f"Quitting, flags: {flags}")
    case ["stop"] if len(command) == 1:   # guard clause
        print("Stopping")
    case _:
        print("Unknown")
# Output: Quitting, flags: ['--force']

# Constructor-like expressions test type at runtime
def process(data):
    match data:
        case str(name):         # succeeds if data is a str
            print(f"String: {name}")
        case int(value):
            print(f"Integer: {value}")
        case _:
            print("Other")

process("hello")  # String: hello
process(42)       # Integer: 42

# Does NOT match non-sequence iterables (e.g., generators)
gen = (x for x in [1, 2, 3])
match gen:
    case [a, b, c]:   # ❌ fails; generator is not a sequence
        ...
```

---

## Slicing
```python
# Assign iterables to a slice of a mutable sequence
nums = [1, 2, 3, 4, 5]
nums[1:4] = [8, 9]          # replace slice with shorter iterable
print(nums)  # [1, 8, 9, 5]

nums[::2] = [0, 0, 0]       # assign to every other element
print(nums)  # [0, 8, 0, 5, 0] (depending on length)

# Works with any iterable on the right-hand side
letters = ['a', 'b', 'c', 'd']
letters[1:3] = (x.upper() for x in ['x', 'y'])
print(letters)  # ['a', 'X', 'Y', 'd']
```
