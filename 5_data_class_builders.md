# Data Class Builders

## No Inheritance for Functionality

```python
# namedtuple and NamedTuple use metaprogramming to create tuple subclasses.
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
print(isinstance(p, tuple))  # True — is a tuple subclass
print(p)                     # Point(x=10, y=20)
```

```python
# @dataclass injects methods without touching the class hierarchy.
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(10, 20)
print(isinstance(p, tuple))  # False — not a tuple subclass
print(p)                     # Point(x=10, y=20)
```

---

## `collections.namedtuple`

```python
from collections import namedtuple

# Enhanced tuple with field names, class name, and useful __repr__
City = namedtuple('City', ['name', 'country', 'population'])
city = City('Tokyo', 'Japan', 13_960_000)

print(city)                # City(name='Tokyo', country='Japan', population=13960000)
print(city.name)           # Tokyo
print(city._asdict())      # {'name': 'Tokyo', 'country': 'Japan', 'population': 13960000}

# Works anywhere a tuple is expected
x, y, z = city
print(x)  # Tokyo
```

---

## `typing.NamedTuple`

```python
from typing import NamedTuple

class City(NamedTuple):
    name: str
    country: str
    population: int

city = City('Tokyo', 'Japan', 13_960_000)
print(city.__annotations__)
# {'name': <class 'str'>, 'country': <class 'str'>, 'population': <class 'int'>}
```

---

## Advanced Field Customisation

```python
from dataclasses import dataclass, field

@dataclass
class Inventory:
    items: list = field(default_factory=list)   # avoid shared mutable default
    price: float = field(default=0.0, metadata={'unit': 'USD'})
    on_hold: bool = field(default=False, init=False, repr=False)

inv = Inventory(items=['pen'], price=2.5)
print(inv)  # Inventory(items=['pen'], price=2.5)
# on_hold is not in __init__ and not in repr
```

---

## `__post_init__` for Custom Initialisation

```python
@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)

    def __post_init__(self):
        self.area = self.width * self.height

r = Rectangle(3, 4)
print(r)  # Rectangle(width=3, height=4, area=12)
```

---

## `typing.ClassVar` – Class Attributes

```python
from typing import ClassVar

@dataclass
class User:
    MIN_AGE: ClassVar[int] = 18   # class attribute, not an instance field
    name: str
    age: int

u1 = User('Alice', 30)
u2 = User('Bob', 16)
print(User.MIN_AGE)   # 18
print(u1.MIN_AGE)     # 18 (accessible via instance but same shared value)
# MIN_AGE is not in __init__, __repr__, or __eq__
```

---

## `InitVar` – Init‑Only Parameters

```python
from dataclasses import dataclass, InitVar

@dataclass
class Person:
    name: str
    birth_year: InitVar[int]
    age: int = field(init=False)

    def __post_init__(self, birth_year):
        self.age = 2024 - birth_year

p = Person('Alice', 1990)
print(p)         # Person(name='Alice', age=34)
# birth_year is not stored as an attribute
```

---

## `__match_args__` for Positional Pattern Matching

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(3, 4)
match p:
    case Point(x, y):
        print(f"{x=}, {y=}")   # x=3, y=4
# __match_args__ = ('x', 'y') is auto‑generated
```

```python
# Customise __match_args__ to hide internal fields
@dataclass
class Person:
    __match_args__ = ('name',)
    name: str
    age: int
    _id: str = field(repr=False)

p = Person('Alice', 30, _id='XYZ')
match p:
    case Person(name):
        print(f"Matched {name}")  # Matched Alice
```

---

## Dataclasses as a Code Smell

```python
# ❌ Dataclass with no behaviour — logic may be scattered elsewhere
@dataclass
class Order:
    id: str
    items: list
    total: float = 0.0

# Total calculation lives outside the class
def calculate_total(order):
    return sum(item.price for item in order.items)

# ✅ Logic should live with the data that it operates on
@dataclass
class Order:
    id: str
    items: list

    @property
    def total(self):
        return sum(item.price for item in self.items)
```
