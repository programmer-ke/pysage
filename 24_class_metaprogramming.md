### Slide 1 – Special Class Attributes

Python stores several useful attributes on every class.

```python
class A:
    pass

class B(A):
    pass

class C(B):
    pass

print(B.__bases__)              # (A,)            – tuple of direct base classes
print(C.__qualname__)           # 'C'             – but inside a module it would be 'module.C'
print(A.__subclasses__())       # [B]             – list of immediate subclasses
print(C.mro())                  # [C, B, A, object] – method resolution order
```

- `__bases__` – direct parents.
- `__qualname__` – fully qualified name (useful in nested classes).
- `__subclasses__()` – live list of sub‑classes (updates as new classes are defined).
- `mro()` – returns the linearisation of the inheritance graph; can be overridden for exotic class hierarchies.

---

### Slide 2 – `type` as a Class Factory

When called with **three arguments**, `type` dynamically creates a new class.  
The arguments are: `name`, `bases` (tuple), and `dict` (mapping of attributes).

```python
# Equivalent to:
# class Person:
#     species = 'human'
#     def greet(self):
#         return 'Hello'

Person = type('Person', (object,), {
    'species': 'human',
    'greet': lambda self: 'Hello'
})

p = Person()
print(p.greet())          # Hello
print(Person.species)     # human
print(type(p))            # <class '__main__.Person'>
```

Optional keyword arguments to `type()` are forwarded to `__init_subclass__` of any base class that defines it.

```python
class Base:
    def __init_subclass__(cls, **kwargs):
        print("Keyword args:", kwargs)

# Passing keyword to type
Child = type('Child', (Base,), {}, extra='info')   # prints Keyword args: {'extra': 'info'}
```

---

### Slide 3 – `type` Is a Metaclass

`type` is the default metaclass – it builds all classes.  
In type hints, `type` means any class object.

```python
class A:
    pass

print(type(A))             # <class 'type'>
print(isinstance(A, type)) # True
```

Narrowing: `type[tuple]` means "a class that is a subclass of `tuple`".

```python
def make_subclass(base: type[tuple]) -> type[tuple]:
    return type('Sub', (base,), {})
```

---

### Slide 4 – Retrieving Annotations

In Python ≥3.10, `inspect.get_annotations` is the modern way to fetch annotations; it replaced `typing.get_type_hints` in many cases.

```python
import inspect

class Point:
    x: int
    y: int

print(inspect.get_annotations(Point))
# {'x': int, 'y': int}
```

---

### Slide 5 – `__init_subclass__` Hook

`__init_subclass__` is called **automatically** when a class inherits from the one that defines it, at **import time**. It allows you to customise subclasses without a metaclass.

```python
class Validated:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        # Example: ensure a specific attribute exists
        if not hasattr(cls, 'required_field'):
            raise TypeError(f"{cls.__name__} must define 'required_field'")

class Good(Validated):
    required_field = 42   # OK

class Bad(Validated):     # TypeError raised at definition time
    pass
```

---

### Slide 6 – `setattr` Triggers `__setattr__`

When you use the `setattr` built‑in, it invokes the object’s `__setattr__` method. This is important for classes that override attribute assignment.

```python
class Watcher:
    def __setattr__(self, name, value):
        print(f"Setting {name} = {value}")
        super().__setattr__(name, value)

w = Watcher()
setattr(w, 'x', 10)   # prints Setting x = 10
```

---

### Slide 7 – `__slots__` Cannot Be Set via `__init_subclass__`

`__slots__` is a class **builder** directive that must be present **before** the class is created (i.e., passed to `type.__new__`).  
`__init_subclass__` runs **after** the class is built, so you cannot add or modify `__slots__` there.

```python
class Base:
    def __init_subclass__(cls, **kwargs):
        # This has NO effect – the class is already built
        cls.__slots__ = ('x',)

# Attempting to do this results in a class without __slots__
class Child(Base):
    pass

c = Child()
c.y = 5   # works, __dict__ is present
```

To add `__slots__` dynamically, you must intercept earlier, e.g., in a metaclass `__new__`.

---

### Slide 8 – Class Decorators

A class decorator is a callable that receives a class and returns a (possibly modified) class.  
It runs after `__init_subclass__` and can be used to avoid metaclass conflicts or heavy inheritance.

```python
def add_repr(cls):
    cls.__repr__ = lambda self: f"{self.__class__.__name__}({vars(self)})"
    return cls

@add_repr
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p)          # Point({'x': 1, 'y': 2})
```

Class decorators are often simpler than metaclasses for modifying a class after creation.

---

### Slide 9 – Import‑Time Class Building

At import time, Python:
- parses the source,
- compiles bytecode,
- executes the module’s top‑level code – that’s when class bodies run.

Thus, **class metaprogramming happens at import time**, not at runtime after the fact.

```python
# This code runs immediately when the module is imported
print("Building class X...")
class X:
    print("Inside class X body")
    def method(self):
        pass
print("Class X built.")
```

---

### Slide 10 – The `object`–`type` Relationship

`type` is a subclass of `object`, and `object` is an **instance** of `type`.  
Every class is an instance of `type`, but only metaclasses are **subclasses** of `type`.

```python
print(isinstance(object, type))    # True – object is created by type
print(issubclass(type, object))    # True – type inherits from object
print(issubclass(type, type))      # True
```

This circular arrangement is deliberately hard‑coded into Python.

---

### Slide 11 – Metaclass `__new__`

When a class with a custom metaclass is defined, its metaclass’s `__new__` is called with four arguments:
- `mcs` – the metaclass
- `name` – the class name
- `bases` – tuple of base classes
- `namespace` – the class’s attribute dictionary

You can modify the namespace before handing it to `type.__new__`, and modify the resulting class afterwards.

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print("Creating class:", name)
        # Add a counter to every class
        namespace['counter'] = 0
        cls = super().__new__(mcs, name, bases, namespace)
        cls._created_at = "just now"
        return cls

class MyClass(metaclass=Meta):
    pass

print(MyClass.counter)       # 0
print(MyClass._created_at)   # just now
```

---

### Slide 12 – Order of Operations During Class Creation

1. Python calls the metaclass `__prepare__` to get the namespace mapping.
2. The class body is executed, populating the namespace.
3. The metaclass `__new__` is called with that namespace – this is where the class object is built (usually via `type.__new__`).
4. After `__new__` returns, `__init_subclass__` is called on any superclass that defines it.
5. Finally, any class decorator is applied.

```python
def class_decorator(cls):
    print("Decorator for", cls.__name__)
    return cls

class Base:
    def __init_subclass__(cls, **kwargs):
        print("__init_subclass__ for", cls.__name__)

class Meta(type):
    def __prepare__(name, bases):
        print("__prepare__")
        return {}
    def __new__(mcs, name, bases, namespace):
        print("__new__")
        return super().__new__(mcs, name, bases, namespace)
    def __init__(cls, name, bases, namespace):
        print("__init__")
        super().__init__(name, bases, namespace)

@class_decorator
class Derived(Base, metaclass=Meta):
    x = 5

# Output order:
# __prepare__
# __new__
# __init__
# __init_subclass__ for Derived
# Decorator for Derived
```

---

### Slide 13 – `__slots__` Must Be Added Before `type.__new__`

`__slots__` is interpreted by `type.__new__` at class creation.  
You must put it into the namespace **before** calling the super `__new__`.

```python
class SlotMeta(type):
    def __new__(mcs, name, bases, namespace):
        if 'slots' in namespace:
            namespace['__slots__'] = tuple(namespace['slots'])
            del namespace['slots']
        return super().__new__(mcs, name, bases, namespace)

class SlotClass(metaclass=SlotMeta):
    slots = ['x', 'y']

s = SlotClass()
s.x = 1
s.z = 2   # AttributeError
```

---

### Slide 14 – `__prepare__` – Custom Namespace

`__prepare__` returns the mapping that will hold the class attributes.  
By default it’s a plain `dict`, but you can return an `OrderedDict` or a custom mapping to control attribute order or semantics.

```python
from collections import OrderedDict

class OrderedMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases):
        return OrderedDict()

class MyClass(metaclass=OrderedMeta):
    z = 3
    a = 1
    m = 2

print([name for name in MyClass.__dict__ if not name.startswith('__')])
# ['z', 'a', 'm'] – preserved definition order
```

---

### Slide 15 – Customising `repr()` of Classes via Metaclass

By defining `__repr__` on a metaclass, you control what `repr(SomeClass)` returns.

```python
class ReprMeta(type):
    def __repr__(cls):
        return f"<class {cls.__qualname__}>"

class A(metaclass=ReprMeta):
    pass

print(repr(A))   # <class A>
```

---

### Slide 16 – `__slots__` and Descriptors: No Name Clash

When a class uses `__slots__`, it cannot have both a class attribute and an instance attribute with the same name.  
Descriptors therefore must use **a different storage name** (e.g., prefix with `_`).

```python
class Desc:
    def __set_name__(self, owner, name):
        self.storage_name = '_' + name

    def __get__(self, instance, owner):
        return getattr(instance, self.storage_name, None)

    def __set__(self, instance, value):
        setattr(instance, self.storage_name, value)

class SlotDemo:
    __slots__ = ('_x',)   # storage slot
    x = Desc()            # descriptor

s = SlotDemo()
s.x = 10
print(s.x)    # 10
```

If you used `__slots__ = ('x',)` the class creation would fail because `x` is already a class attribute (the descriptor).

---

### Slide 17 – Features That Reduce the Need for Metaclasses

Python now offers three lighter mechanisms that cover many metaclass use cases:

- **`__set_name__`** – descriptors know their own name automatically.
- **Class decorators** – modify classes after creation without inheritance or metaclass.
- **`__init_subclass__`** – hook into subclass creation without a metaclass.

```python
# __init_subclass__ example again
class Base:
    def __init_subclass__(cls):
        super().__init_subclass__()
        cls.created = True

class Child(Base):
    pass

print(Child.created)   # True
```

---

### Slide 18 – One Metaclass Only (Multiple Inheritance Problem)

A class can have only **one** metaclass.  
If you try to inherit from two classes that each have different metaclasses, Python raises a `TypeError`.

```python
class MetaA(type): pass
class MetaB(type): pass

class A(metaclass=MetaA): pass
class B(metaclass=MetaB): pass

# This fails:
# class C(A, B): pass   # TypeError: metaclass conflict
```

The solution is to create a **combined metaclass** that inherits from both.

```python
class Combined(MetaA, MetaB): pass

class C(A, B, metaclass=Combined):
    pass   # OK
```

---

### Slide 19 – Metaclasses as Implementation Details

Metaclasses are powerful; they should be hidden from end users if possible.  
Think of them as the "plumbing" that makes a framework work – not something users of your classes need to understand.

```python
# Users see this:
class UserModel(BaseModel):
    name: str
    age: int

# Behind the scenes, BaseModel may use a metaclass
class BaseModel(metaclass=ModelMeta):
    ...
```

---

### Slide 20 – Reserve Powerful Features for Frameworks

Features like ABCs, operator overloading, descriptors, class decorators, and metaclasses add complexity.  
Use them **only when a clear need arises**, and consider extracting them into a library so that application code stays simple.

---

### Slide 21 – Four Interception Points in Metaclasses

1. **`__prepare__`** – provides the attribute dictionary (allows custom mapping).
2. **`__new__`** – called *before* the class object is created; can modify namespace and control class creation.
3. **`__init__`** – called *after* the class is created; like `__init__` for instances, but for classes.
4. **`__call__`** – controls what happens when the class is called (i.e., when you create an instance). Override this to customise instance creation (like a singleton pattern).

```python
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Singleton(metaclass=SingletonMeta):
    pass

a = Singleton()
b = Singleton()
print(a is b)   # True
```

---

**That covers every point from the Class Metaprogramming section** – from special attributes and `type` as a factory to metaclass hooks and when to use them responsibly.
