# Functions as First‑Class Objects

## Higher‑Order Functions

```python
# A higher‑order function takes a function as argument or returns one
def apply_twice(f, x):
    return f(f(x))

def square(n):
    return n * n

print(apply_twice(square, 3))  # 81 → square(square(3))
```

```python
# Returning a function
def make_multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply

double = make_multiplier(2)
print(double(5))  # 10
```

---

## The `__doc__` Attribute

```python
def greet(name):
    """Return a friendly greeting."""
    return f"Hello, {name}!"

print(greet.__doc__)  # Return a friendly greeting.
help(greet)           # uses __doc__ to generate help text
```

---

## Classic Higher‑Order Functions

```python
# map, filter, reduce
from functools import reduce

nums = [1, 2, 3, 4]
print(list(map(str, nums)))              # ['1', '2', '3', '4']
print(list(filter(lambda x: x > 2, nums)))  # [3, 4]
print(reduce(lambda a, b: a * b, nums))  # 24
```

```python
# apply (deprecated) replaced by fn(*args, **kwargs)
def func(a, b):
    return a + b

args = (3, 4)
print(func(*args))        # 7 — modern Python style
```

---

## Useful Reducing Built‑ins

```python
print(sum([1, 2, 3]))    # 6
print(all([True, True])) # True
print(any([False, True]))# True
```

---

## Anonymous Functions (`lambda`)

```python
# Best used as arguments to higher-order functions
pairs = [(1, 'one'), (3, 'three'), (2, 'two')]
pairs.sort(key=lambda p: p[0])
print(pairs)  # [(1, 'one'), (2, 'two'), (3, 'three')]

# Prefer named functions elsewhere for clarity
def get_age(person):
    return person.age

# better than lambda p: p.age in non‑trivial contexts
```

---

## Callable Objects

```python
class Counter:
    def __init__(self):
        self.count = 0

    def __call__(self):
        self.count += 1
        return self.count

counter = Counter()
print(counter())  # 1
print(counter())  # 2 — state persists across calls

# Useful for decorators and function‑like objects
```

---

## Keyword‑Only Arguments

```python
def config(*, host, port):
    print(f"host={host}, port={port}")

# Keyword‑only: must be passed by name
config(host='localhost', port=8080)
# config('localhost', 8080)  ❌ TypeError
```

---

## Position‑Only Arguments

```python
def multiply(a, b, /):
    return a * b

# Position‑only: cannot be passed by keyword
print(multiply(3, 4))   # 12
# print(multiply(a=3, b=4))  ❌ TypeError
```

---

## The `operator` Module

```python
from operator import add, mul, gt

# Function equivalents of operators
print(add(3, 4))   # 7
print(mul(3, 4))   # 12
print(gt(5, 2))    # True

# Avoid trivial lambdas
from functools import reduce
print(reduce(mul, [1, 2, 3, 4]))  # 24 — cleaner than lambda a,b: a*b
```

---

## `operator.itemgetter`

```python
from operator import itemgetter

data = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}]
get_name = itemgetter('name')
print(list(map(get_name, data)))  # ['Alice', 'Bob']

# Works with sequences, mappings, or anything with __getitem__
students = [('Alice', 90), ('Bob', 85)]
get_second = itemgetter(1)
print(list(map(get_second, students)))  # [90, 85]
```

---

## `operator.attrgetter` and `operator.methodcaller`

```python
from operator import attrgetter, methodcaller

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    def greet(self, greeting):
        return f"{greeting}, {self.name}!"

people = [Person('Alice', 30), Person('Bob', 25)]
get_age = attrgetter('age')
print(list(map(get_age, people)))  # [30, 25]

greeter = methodcaller('greet', 'Hi')
print(list(map(greeter, people)))
# ['Hi, Alice!', 'Hi, Bob!']
```

---

## `functools.partial`

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

# Pre‑bind arguments to create a simpler callable
square = partial(power, exponent=2)
cube   = partial(power, exponent=3)

print(square(5))  # 25
print(cube(5))    # 125

# Useful for adapting functions to APIs that expect fewer arguments
```
