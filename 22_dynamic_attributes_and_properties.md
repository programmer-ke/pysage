### Slide 1 – `__getattr__` only fires when the attribute is missing

`__getattr__` is a fallback method. Python calls it **only after** normal attribute lookup (instance, class, inheritance) fails. It is **not** called if the attribute exists somewhere in the MRO.

```python
class Missing:
    def __getattr__(self, name):
        return f"'{name}' is not here!"

obj = Missing()
print(obj.x)        # calls __getattr__ -> "'x' is not here!"
obj.y = 5
print(obj.y)        # 5 (normal access, __getattr__ skipped)
```

---

### Slide 2 – `__dir__` helps interactive tools

Implement `__dir__` to control what `dir()` returns – and what appears in IDE autocompletion / REPL suggestions.

```python
class Person:
    def __init__(self, name):
        self._name = name

    def __dir__(self):
        # Show public attribute plus custom listing
        return ['name', '_name']

p = Person("Carol")
print(dir(p))       # ['_name', 'name']
```

---

### Slide 3 – `keyword.iskeyword` guards against keyword conflicts

If you dynamically generate attribute names, make sure they aren't Python keywords. `keyword.iskeyword` checks this.

```python
import keyword

label = "class"
if keyword.iskeyword(label):
    print(f"'{label}' is a keyword, use a different name")
```

---

### Slide 4 – Construction vs Initialisation: `__new__` and `__init__`

- `__new__` is the **constructor** – it creates the actual instance (before `self` exists).
- `__init__` is the **initialiser** – it configures the instance after creation.
- If `__new__` returns an object of a different class, `__init__` is **not** called.
- The first argument to `__new__` is the **class**; the remaining arguments are the same as for `__init__`.

```python
class Weird:
    def __new__(cls, *args, **kwargs):
        # Instead of creating a Weird instance, return a list
        return [42]

obj = Weird()
print(type(obj))   # <class 'list'>
# __init__ was never called because the returned object is not a Weird
```

---

### Slide 5 – Using `super().__new__` correctly

When you override `__new__`, you normally call `object.__new__(cls)` (or `super().__new__(cls)`). This creates a fresh instance of the **given class**.

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        # Create an instance of the *actual* class (could be a subclass)
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        self.value = value

class MySub(MyClass):
    pass

obj = MySub(10)
print(type(obj))   # <class '__main__.MySub'>
```

---

### Slide 6 – Quick attribute injection via `__dict__` update

You can populate a class with many attributes at once by updating its `__dict__`. This is a low‑level shortcut.

```python
class Config:
    pass

# Add attributes after class creation
Config.__dict__.update({
    'host': 'localhost',
    'port': 8080,
    'debug': True,
})

print(Config.host)   # localhost
```

---

### Slide 7 – Lightweight “Namespace” objects from the standard library

Python provides ready‑made classes that accept keyword arguments and turn them into attributes:

- `types.SimpleNamespace`
- `argparse.Namespace`
- `multiprocessing.managers.Namespace`

```python
from types import SimpleNamespace

person = SimpleNamespace(name="Alice", age=30)
print(person.name)   # Alice
print(person.age)    # 30
```

---

### Slide 8 – Inside a property, don't use the same name (or you'll recurse)

If a property uses the same name as its backing attribute, accessing that name inside the getter/setter causes infinite recursion.  
**Solution**: use `self.__dict__` directly, or a different name (e.g., `_name`).

```python
class Bad:
    @property
    def x(self):
        return self.x   # RecursionError!

# Correct approach
class Good:
    @property
    def x(self):
        return self.__dict__['x']   # bypasses property lookup

    @x.setter
    def x(self, value):
        self.__dict__['x'] = value
```

---

### Slide 9 – Race conditions with handmade property caches

If you cache a property value manually (e.g., in `__dict__`) without synchronisation, concurrent threads may recompute it or see inconsistent states. Use `functools.cached_property` for thread‑safe caching.

```python
# Handmade cache – NOT thread‑safe
class Unsafe:
    @property
    def data(self):
        if '_data' not in self.__dict__:
            # another thread could also be here
            self.__dict__['_data'] = expensive_computation()
        return self.__dict__['_data']
```

---

### Slide 10 – `functools.cached_property` – safe, single‑shot caching

`cached_property` runs only once per instance and stores the result as an instance attribute. If the attribute already exists, it is returned immediately.

```python
from functools import cached_property
import time

class Report:
    @cached_property
    def summary(self):
        time.sleep(2)           # heavy computation
        return "Summary content"

r = Report()
print(r.summary)   # first access: computes and stores
print(r.summary)   # second access: returns cached value instantly
del r.summary      # delete the cached attribute
print(r.summary)   # recomputes
```

---

### Slide 11 – Stacking `@property` on `@functools.cache` – a different caching model

`@functools.cache` works at the function level, not per instance. Combined with `@property`, the cache lives **outside** the instance and depends on function arguments (here, the implicit `self`), so each instance gets its own cached value. The property still runs as a method, but `cache` avoids recomputation for the same `self`.

```python
from functools import cache

class Computer:
    @property
    @cache
    def magic(self):
        print("Computing...")
        return 42

c1 = Computer()
c2 = Computer()
print(c1.magic)   # Computing... 42
print(c1.magic)   # 42 (no recomputation)
print(c2.magic)   # Computing... 42 (different instance)
```

---

### Slide 12 – Custom validation via property setters in `__init__`

You can use the same property setter logic during `__init__` to validate values at instantiation time.

```python
class Temperature:
    def __init__(self, celsius):
        self.celsius = celsius   # calls the setter

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Below absolute zero!")
        self._celsius = value

t = Temperature(20)      # ok
t.celsius = -300         # ValueError raised
```

---

### Slide 13 – The `property` built‑in is a class

`property()` can be constructed with `fget`, `fset`, `fdel`, and `doc`. You can use it without decorators.

```python
class Rectangle:
    def get_width(self):
        return self._width
    def set_width(self, value):
        self._width = value
    def del_width(self):
        del self._width

    width = property(get_width, set_width, del_width, "Width in pixels")

r = Rectangle()
r.width = 100
print(r.width)   # 100
del r.width
```

---

### Slide 14 – Attribute lookup order: class properties before instance dict

When you write `obj.attr`, Python first checks `type(obj).__mro__` for a **data descriptor** (i.e., a property). If found, it uses that **before** looking in the instance’s `__dict__`. This is why you can't shadow a property with an instance attribute of the same name.

```python
class Example:
    @property
    def answer(self):
        return 42

e = Example()
e.__dict__['answer'] = 99   # store a value in instance dict
print(e.answer)              # still 42 (property overrides)
print(e.__dict__['answer'])  # 99 (raw dict access bypasses property)
```

---

### Slide 15 – Property factories using closures

A factory function can return a `property` object whose getter/setter know which name to use via closure, avoiding repetition.

```python
def typed_property(name, expected_type):
    private_name = '_' + name
    @property
    def prop(self):
        return getattr(self, private_name)
    @prop.setter
    def prop(self, value):
        if not isinstance(value, expected_type):
            raise TypeError(f"{name} must be {expected_type}")
        setattr(self, private_name, value)
    return prop

class Person:
    name = typed_property('name', str)
    age = typed_property('age', int)

p = Person()
p.name = "Bob"   # ok
p.age = 30       # ok
p.age = "old"    # TypeError
```

---

### Slide 16 – Using `@property.deleter`

In a class body, `@property.deleter` marks the method that will be called when `del obj.attr` is used on a managed attribute.

```python
class Resource:
    def __init__(self):
        self._handle = open('/dev/null', 'w')

    @property
    def handle(self):
        return self._handle

    @handle.deleter
    def handle(self):
        print("Closing handle")
        self._handle.close()

r = Resource()
del r.handle   # prints "Closing handle", closes file
```

---

### Slide 17 – Where Python looks for special methods

Special methods (e.g., `__getattr__`) are always looked up **on the object’s class**, not the instance. This prevents them from being overridden by instance attributes.

```python
class Demo:
    def __getattr__(self, name):
        return f"missing: {name}"

d = Demo()
d.__getattr__ = lambda self, name: "shadow"   # shadows only as instance attr
print(d.xyz)           # "missing: xyz" – class __getattr__ still used
```

---

### Slide 18 – Attributes that control instance storage: `__dict__` and `__slots__`

- `__dict__` stores all writable attributes.
- `__slots__` replaces `__dict__` with fixed tuple of attribute names, saving memory and preventing new attribute creation.

```python
class SlotDemo:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

s = SlotDemo(1, 2)
s.z = 3   # AttributeError: 'SlotDemo' object has no attribute 'z'
```

---

### Slide 19 – Built‑in functions for attribute introspection

| Function | Description |
|----------|-------------|
| `dir(obj)` | List most attributes (uses `__dir__` if defined) |
| `getattr(obj, name[, default])` | Retrieve attribute safely |
| `hasattr(obj, name)` | Check existence (uses `getattr` internally) |
| `setattr(obj, name, value)` | Set attribute |
| `vars(obj)` | Return `obj.__dict__` (or local scope if no argument) |

```python
class Box:
    def __init__(self):
        self.color = 'blue'

box = Box()
print(hasattr(box, 'color'))   # True
print(getattr(box, 'weight', 0)) # 0 (default)
setattr(box, 'weight', 5)
print(vars(box))               # {'color': 'blue', 'weight': 5}
```

---

### Slide 20 – The special‑method family for attribute access

These methods are invoked by dot notation, `getattr`, `setattr`, etc., but **not** when you access `obj.__dict__` directly. They are also never shadowed by instance attributes.

- `__getattr__` – fallback on missing attribute
- `__getattribute__` – called for **every** attribute lookup (be careful!)
- `__setattr__` – called on **every** attribute assignment
- `__delattr__` – called on `del obj.attr`
- `__dir__` – custom `dir()` output

```python
class Traced:
    def __getattribute__(self, name):
        print(f"Access: {name}")
        return super().__getattribute__(name)

    def __setattr__(self, name, value):
        print(f"Set: {name} = {value}")
        super().__setattr__(name, value)

    def __delattr__(self, name):
        print(f"Delete: {name}")
        super().__delattr__(name)

t = Traced()
t.x = 10        # Set: x = 10
print(t.x)      # Access: x  10
del t.x         # Delete: x
```

---

### Slide 21 – Why properties/descriptors are usually better than `__getattribute__`/`__setattr__`

Overriding `__getattribute__` and `__setattr__` can lead to infinite recursion and subtle bugs. Use properties or descriptors instead for specific attribute behaviour.

```python
# Better: use property for validation
class Point:
    def __init__(self, x):
        self.x = x

    @property
    def x(self):
        return self._x

    @x.setter
    def x(self, value):
        if not isinstance(value, (int, float)):
            raise TypeError("x must be numeric")
        self._x = value
```

---

**That covers every point from the Dynamic Attributes and Properties section.** We saw how Python’s attribute model works from the inside out, how to customise it safely, and when to prefer properties over lower‑level hooks.
