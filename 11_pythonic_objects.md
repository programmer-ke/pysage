# A Pythonic Object

The Python data model encourages making user-defined types behave like built‑ins. The goal is to create objects that feel natural to use—objects that support iteration, formatting, hashing, and other Python idioms without forcing users to learn custom APIs.

---

## 1. Duck Typing Without Inheritance

You don't need to inherit from a special base class to make your objects behave like built‑ins. Just implement the right special methods.

```python
class Card:
    def __init__(self, rank, suit):
        self.rank = rank
        self.suit = suit

    def __repr__(self):
        return f"Card('{self.rank}', '{self.suit}')"

    def __str__(self):
        return f"{self.rank} of {self.suit}"

    def __eq__(self, other):
        return (self.rank, self.suit) == (other.rank, other.suit)

# No inheritance—just works
c1 = Card('A', 'spades')
c2 = Card('A', 'spades')

print(c1)             # A of spades
print(repr(c1))       # Card('A', 'spades')
print(c1 == c2)       # True
print(c1 is c2)       # False
```

By implementing `__repr__`, `__str__`, and `__eq__`, `Card` behaves like a built‑in type without inheriting from anything except `object`.

---

## 2. `classmethod` for Alternative Constructors

`classmethod` receives the class as its first argument (`cls`), making it perfect for defining alternative ways to create instances.

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    def __repr__(self):
        return f"Date({self.year}, {self.month}, {self.day})"

    @classmethod
    def from_iso(cls, iso_string):
        """Alternative constructor: parse 'YYYY-MM-DD'."""
        year, month, day = map(int, iso_string.split('-'))
        return cls(year, month, day)

    @classmethod
    def today(cls):
        """Alternative constructor: create from current date."""
        import datetime
        d = datetime.date.today()
        return cls(d.year, d.month, d.day)

d1 = Date(2025, 1, 15)
d2 = Date.from_iso("2025-01-15")
d3 = Date.today()

print(d1 == d2)  # True (assuming __eq__ is implemented)
```

Each `@classmethod` provides a different path to constructing a `Date`, and because it uses `cls` rather than hardcoding `Date`, it works correctly in subclasses.

---

## 3. `staticmethod` for Functions That Live in a Class

`staticmethod` is for functions that are logically related to the class but don't need access to the instance (`self`) or the class (`cls`).

```python
class StringUtils:
    @staticmethod
    def is_palindrome(s):
        """Check if a string reads the same forwards and backwards."""
        s = s.lower().replace(' ', '')
        return s == s[::-1]

    @staticmethod
    def word_count(s):
        return len(s.split())

print(StringUtils.is_palindrome("A man a plan a canal Panama"))  # True
print(StringUtils.word_count("hello world"))                     # 2
```

These are just regular functions that happen to be namespaced inside the class. No `self` or `cls` is passed.

---

## 4. `__format__` for Custom Formatting

f‑strings, the built‑in `format()`, and `str.format()` all delegate to the object's `__format__(format_spec)` method. If you don't define it, the inherited `object.__format__` simply calls `str(self)`.

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

    def __format__(self, format_spec):
        """Support format specs: 'p' for polar, default for cartesian."""
        import math
        if format_spec == 'p':
            r = math.hypot(self.x, self.y)
            theta = math.atan2(self.y, self.x)
            return f"(r={r:.2f}, θ={theta:.2f})"
        else:
            return f"({self.x}, {self.y})"

v = Vector(3, 4)

print(f"{v}")       # (3, 4)
print(f"{v:p}")     # (r=5.00, θ=0.93)
print(format(v, 'p'))  # Same thing via format()
```

By implementing `__format__`, your objects participate fully in Python's string formatting ecosystem.

---

## 5. Making Objects Hashable with `__hash__` and `__eq__`

To use an object as a dictionary key or in a set, it must be hashable. This requires implementing **both** `__hash__` and `__eq__`. Use `property` to make attributes read‑only, preventing accidental modification that would break hashing.

```python
class Point:
    def __init__(self, x, y):
        self._x = x
        self._y = y

    @property
    def x(self):
        """Read-only x coordinate."""
        return self._x

    @property
    def y(self):
        """Read-only y coordinate."""
        return self._y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def __repr__(self):
        return f"Point({self.x}, {self.y})"

# Now Points work as dictionary keys
p1 = Point(1, 2)
p2 = Point(1, 2)
p3 = Point(3, 4)

d = {p1: "origin", p3: "destination"}
print(d[p2])  # "origin" — p1 and p2 are equal and have the same hash

# p1.x = 5  # AttributeError: can't set attribute
```

**Rules for hashability:**
- Objects that compare equal must have the same hash.
- The hash must not change over the object's lifetime (hence read‑only attributes).
- If you define `__eq__` without `__hash__`, the object becomes unhashable.

---

## 6. Simplicity for Apps, Completeness for Libraries

For application code, keep classes simple. Only implement what you need. But for **libraries**, be more thorough—you can't predict how users will use your objects.

```python
# Application code: just enough
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

# Library code: be pythonic
class LibraryUser:
    def __init__(self, name, email):
        self._name = name
        self._email = email

    @property
    def name(self):
        return self._name

    def __repr__(self):
        return f"LibraryUser({self.name!r}, {self._email!r})"

    def __eq__(self, other):
        if not isinstance(other, LibraryUser):
            return NotImplemented
        return self._email == other._email

    def __hash__(self):
        return hash(self._email)
```

When writing a library, anticipate that users will want to sort, hash, compare, serialize, and introspect your objects.

---

## 7. Name Mangling with Double Underscores

Prefixing an attribute with `__` (two underscores, not ending with two underscores) triggers **name mangling**: Python rewrites the name to include the class name, making accidental clobbering by subclasses less likely.

```python
class Base:
    def __init__(self):
        self.__secret = "hidden"  # Actually stored as _Base__secret
        self._internal = "convention"  # No mangling, just a convention

class Sub(Base):
    def __init__(self):
        super().__init__()
        self.__secret = "overridden?"  # This is _Sub__secret, a different attribute!

    def reveal(self):
        print(f"_Sub__secret: {self.__secret}")
        print(f"_Base__secret via mangled name: {self._Base__secret}")

s = Sub()
s.reveal()
# _Sub__secret: overridden?
# _Base__secret via mangled name: hidden

print(s.__dict__)
# {'_Base__secret': 'hidden', '_internal': 'convention', '_Sub__secret': 'overridden?'}
```

**Important:** Name mangling is **not for security**. It's only to avoid accidental collisions in subclasses. Many developers prefer a single underscore (`_internal`) as a convention meaning "treat this as non‑public."

---

## 8. `__slots__` for Memory Efficiency

By default, each instance stores its attributes in a `__dict__`, a hash table. `__slots__` replaces `__dict__` with a fixed, compact array, saving memory when you have many instances.

```python
import sys

class WithoutSlots:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class WithSlots:
    __slots__ = ('x', 'y', 'z')
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

wos = WithoutSlots(1, 2, 3)
ws = WithSlots(1, 2, 3)

print(hasattr(wos, '__dict__'))  # True
print(hasattr(ws, '__dict__'))   # False

# Memory comparison
many_wos = [WithoutSlots(1, 2, 3) for _ in range(1_000_000)]
# ~160 MB

many_ws = [WithSlots(1, 2, 3) for _ in range(1_000_000)]
# ~80 MB — roughly half the memory!
```

**Rules and caveats:**
- Extra attributes not listed in `__slots__` will raise `AttributeError`.
- Subclasses inherit `__dict__` unless they define their own `__slots__`.
- To support weak references, include `'__weakref__'` in `__slots__`.

```python
class MyClass:
    __slots__ = ('x', 'y', '__weakref__')  # Allows weak references

import weakref
obj = MyClass()
ref = weakref.ref(obj)  # Works!
```

---

## 9. Class Attributes as Public Configuration

Because class attributes are public, it's common to create subclasses solely to change them. This is a lightweight form of configuration.

```python
class ApiClient:
    base_url = "https://api.example.com"
    timeout = 30
    max_retries = 3

    def request(self, path):
        full_url = f"{self.base_url}/{path}"
        print(f"Requesting {full_url} "
              f"(timeout={self.timeout}, retries={self.max_retries})")
        # ... actual request logic ...

# Override class attributes in a subclass
class StagingApiClient(ApiClient):
    base_url = "https://staging-api.example.com"
    timeout = 60

class ProductionApiClient(ApiClient):
    base_url = "https://prod-api.example.com"
    max_retries = 5

prod = ProductionApiClient()
staging = StagingApiClient()

prod.request("users")    # Requesting https://prod-api.example.com/users (timeout=30, retries=5)
staging.request("users") # Requesting https://staging-api.example.com/users (timeout=60, retries=3)
```

You can also override class attributes on an instance:

```python
client = ApiClient()
client.timeout = 10  # Overrides for this instance only
client.request("data")
# Requesting https://api.example.com/data (timeout=10, retries=3)
```

This pattern is used extensively in Django settings, Flask configuration, and many other frameworks.

---

## Summary

| Topic | Key Idea |
|---|---|
| **Duck typing** | Implement special methods; no inheritance needed |
| **`@classmethod`** | Alternative constructors (e.g., `from_iso`, `today`) |
| **`@staticmethod`** | Utility functions namespaced in a class |
| **`__format__`** | Control output of f‑strings and `format()` |
| **`__hash__` / `__eq__`** | Make objects usable as dict keys / set members |
| **Property** | Read‑only attributes to preserve hash consistency |
| **Simplicity vs. completeness** | Apps: keep it simple. Libraries: be pythonic |
| **Name mangling** | `__attr` → `_ClassName__attr`; prevents accidental clobbering |
| **`__slots__`** | Saves memory; rejects extra attributes; supports `__weakref__` |
| **Class attrs as config** | Subclass or override class attributes for configuration |

Building pythonic objects isn't about following rigid rules—it's about making your types feel natural and predictable to other Python programmers.
