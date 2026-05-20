---
## The Python Data Model

### Formalized Interfaces for Language Building Blocks

```python
# Sequences
class BookShelf:
    def __init__(self, books):
        self._books = list(books)
    
    def __getitem__(self, index):
        return self._books[index]
    
    def __len__(self):
        return len(self._books)

shelf = BookShelf(['1984', 'Dune', 'Neuromancer'])

# All this works automatically:
len(shelf)              # 3
shelf[0]                # '1984'
shelf[-1]               # 'Neuromancer'
for book in shelf:      # iteration
    print(book)
'Dune' in shelf         # True
sorted(shelf)           # ['1984', 'Dune', 'Neuromancer']
```

### Functions

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor
    
    def __call__(self, x):
        return x * self.factor

double = Multiplier(2)
double(5)  # 10
```

### Iterators

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

for n in Countdown(3):
    print(n)  # 3, 2, 1
```

### Context Managers

```python
class ManagedFile:
    def __init__(self, filename, mode='r'):
        self.filename = filename
        self.mode = mode
    
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()

with ManagedFile('data.txt', 'w') as f:
    f.write('Hello')
# file auto-closed
```

---

### No Custom Methods to Learn

```python
# Users interact through familiar Python syntax
shelf = BookShelf(['A', 'B', 'C'])

# Not: shelf.get_book(0) or shelf.book_count()
# But:
shelf[0]     # standard indexing
len(shelf)   # standard built-in
```

---

### Composable Interfaces (No Inheritance Required)

```python
class Team:
    def __init__(self, members):
        self._members = list(members)
    
    def __getitem__(self, index):
        return self._members[index]
    
    def __len__(self):
        return len(self._members)

team = Team(['Alice', 'Bob', 'Charlie'])

# Gains all this without inheriting from list:
list(team)                    # ['Alice', 'Bob', 'Charlie']
team[1:]                      # ['Bob', 'Charlie']
reversed(team)                # <reversed object>
max(team)                     # 'Charlie'
'Alice' in team               # True
```

---

### `__str__` vs `__repr__`

```python
from datetime import datetime

class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __repr__(self):
        return f'Point({self.x}, {self.y})'
    
    def __str__(self):
        return f'({self.x}, {self.y})'

p = Point(3, 4)

repr(p)   # 'Point(3, 4)'  — unambiguous, recreatable
str(p)    # '(3, 4)'        — user-friendly
print(p)  # (3, 4)          — print uses str()
p          # Point(3, 4)    — REPL uses repr()
```

### When `__str__` is Missing

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __repr__(self):
        return f'Vector({self.x}, {self.y})'
    # No __str__ defined

v = Vector(1, 2)
str(v)   # 'Vector(1, 2)'  — falls back to __repr__
print(v) # Vector(1, 2)
```

---

### Language Constructs Call Special Methods

```python
class Number:
    def __init__(self, value):
        self.value = value
    
    def __add__(self, other):
        return Number(self.value + other.value)
    
    def __bool__(self):
        return self.value != 0
    
    def __abs__(self):
        return Number(abs(self.value))

a = Number(5)
b = Number(-3)

# We write this:
c = a + b          # calls __add__
if a:              # calls __bool__
    print('True')
d = abs(b)         # calls __abs__

# Not this:
# c = a.__add__(b)
# if a.__bool__():
# d = b.__abs__()
```

```python
# Even basic operations use special methods
x = [1, 2, 3]
len(x)        # calls x.__len__()
x[0]          # calls x.__getitem__(0)
x[0] = 10     # calls x.__setitem__(0, 10)
del x[0]      # calls x.__delitem__(0)
```

---

### Summary

| Special Method | Triggered By |
|----------------|-------------|
| `__getitem__`  | `obj[key]`, iteration, `in` |
| `__len__`      | `len(obj)`, truthiness |
| `__call__`     | `obj()` |
| `__iter__`     | `for x in obj`, `iter(obj)` |
| `__next__`     | `next(obj)` |
| `__enter__`    | `with obj:` |
| `__exit__`     | exiting `with` block |
| `__repr__`     | `repr(obj)`, REPL display |
| `__str__`      | `str(obj)`, `print(obj)` |
| `__add__`      | `obj + other` |
| `__bool__`     | `if obj:`, `bool(obj)` |
