### Slide 1 – The Descriptor Protocol

A descriptor is any class that implements `__get__`, `__set__`, or `__delete__`.  
No inheritance is required – it's a pure protocol.  
The built‑in `property` is a full descriptor (all three methods).  
Partial implementations are common (e.g., only `__get__` for a read‑only descriptor).

```python
class SimpleDescriptor:
    def __get__(self, instance, owner):
        return 42

class MyClass:
    attr = SimpleDescriptor()

obj = MyClass()
print(obj.attr)   # 42 – descriptor's __get__ is called
```

---

### Slide 2 – Descriptors Store Values in `__dict__` Directly

To avoid infinite recursion, descriptors **must** read/write the managed instance’s `__dict__` directly, not via `setattr` or dot notation.

```python
class Reusable:
    def __set__(self, instance, value):
        # BAD: instance.attr = value  → infinite recursion
        instance.__dict__['attr'] = value   # correct

    def __get__(self, instance, owner):
        return instance.__dict__.get('attr', None)
```

---

### Slide 3 – The `__get__` Signature

`__get__(self, instance, owner)` receives:
- `self` – the descriptor instance
- `instance` – the managed object (or `None` when accessed via the class)
- `owner` – the class of the managed object

```python
class Desc:
    def __get__(self, instance, owner):
        print(f"self={self}, instance={instance}, owner={owner}")
        return "result"

class Demo:
    attr = Desc()

d = Demo()
print(d.attr)        # instance = d
print(Demo.attr)     # instance = None
```

---

### Slide 4 – Class‑Level Access Returns the Descriptor Itself

When accessed via the class (e.g., `MyClass.attr`), `instance` is `None`.  
By convention, the descriptor returns `self` to allow introspection.

```python
class Desc:
    def __get__(self, instance, owner):
        if instance is None:
            return self          # class access
        return f"value for {instance}"

class Demo:
    attr = Desc()

print(Demo.attr)      # <__main__.Desc object ...>
print(Demo().attr)    # value for <__main__.Demo object ...>
```

---

### Slide 5 – One Descriptor Instance Shared by All Instances

A descriptor is created **once** when the class is defined.  
All instances of that class share the same descriptor object.  
Any state stored on the descriptor itself is shared across all instances.

```python
class SharedCounter:
    def __init__(self):
        self.count = 0
    def __get__(self, instance, owner):
        self.count += 1
        return self.count

class A:
    x = SharedCounter()
class B:
    x = SharedCounter()

a1, a2 = A(), A()
print(a1.x, a2.x)   # 1 2  (shared across A instances)
b = B()
print(b.x)          # 1  (separate descriptor for B)
```

---

### Slide 6 – `__set_name__` Automates Storage Key

`__set_name__(self, owner, name)` is called when the class is created.  
It tells the descriptor the attribute name it was assigned to.  
This lets you avoid hard‑coding the storage key in `__dict__`.

```python
class NamedDesc:
    def __set_name__(self, owner, name):
        self.storage_name = f"_{name}"

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.storage_name, None)

    def __set__(self, instance, value):
        instance.__dict__[self.storage_name] = value

class Person:
    name = NamedDesc()
    age = NamedDesc()

p = Person()
p.name = "Alice"
p.age = 30
print(p.name, p.age)   # Alice 30
```

---

### Slide 7 – Overriding vs. Non‑Overriding Descriptors

- **Overriding descriptor**: defines `__set__` (even if it only raises an error).  
  It takes precedence over the instance’s `__dict__`.  
  Example: `property`.

- **Non‑overriding descriptor**: does **not** define `__set__`.  
  An instance attribute of the same name **shadows** the descriptor.

```python
class Overriding:
    def __get__(self, instance, owner):
        return "overriding"
    def __set__(self, instance, value):
        pass   # even a no-op makes it overriding

class NonOverriding:
    def __get__(self, instance, owner):
        return "non-overriding"

class Demo:
    over = Overriding()
    non  = NonOverriding()

d = Demo()
d.__dict__['over'] = 99
d.__dict__['non']  = 99
print(d.over)   # "overriding" (descriptor wins)
print(d.non)    # 99 (instance dict shadows descriptor)
```

---

### Slide 8 – Descriptors Cannot Control Class‑Attribute Assignment

A descriptor’s `__set__` only intercepts **instance** attribute assignment.  
Setting the attribute on the **class** itself replaces the descriptor.  
To control class‑level assignment, you must attach the descriptor to a **metaclass**.

```python
class MyDesc:
    def __set__(self, instance, value):
        print("instance set")

class Demo:
    attr = MyDesc()

Demo.attr = 5          # replaces the descriptor entirely
print(Demo.attr)       # 5 (no longer a descriptor)
```

---

### Slide 9 – Functions Are Descriptors (Bound Methods)

User‑defined functions implement `__get__`.  
When accessed via an instance, they return a **bound method** (the function with `self` already filled in).  
When accessed via the class, they return the plain function.

```python
class Demo:
    def method(self):
        return "hello"

d = Demo()
print(d.method)      # <bound method Demo.method of <...>>
print(Demo.method)   # <function Demo.method at ...>
print(d.method())    # "hello"
```

---

### Slide 10 – Usage Tips: Prefer `property` for Simplicity

`property` is a ready‑made descriptor that covers most needs.  
Use it unless you need the extra flexibility of a custom descriptor.

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("negative radius")
        self._radius = value
```

---

### Slide 11 – Read‑Only Descriptors Need a `__set__` That Raises

A descriptor without `__set__` is non‑overriding and can be shadowed.  
To make a truly read‑only descriptor, define `__set__` that raises `AttributeError`.

```python
class ReadOnly:
    def __init__(self, value):
        self.value = value
    def __get__(self, instance, owner):
        return self.value
    def __set__(self, instance, value):
        raise AttributeError("read-only attribute")

class Demo:
    x = ReadOnly(10)

d = Demo()
print(d.x)   # 10
d.x = 20     # AttributeError
```

---

### Slide 12 – Validation Descriptors Need `__set__`

To validate values on assignment, implement `__set__` with the check.

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        return instance.__dict__.get(self.name, 0)
    def __set__(self, instance, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive")
        instance.__dict__[self.name] = value

class Item:
    price = Positive()
    quantity = Positive()

i = Item()
i.price = 10
i.quantity = 5
i.price = -1   # ValueError
```

---

### Slide 13 – Caching with Only `__get__`

A descriptor with only `__get__` can compute a value once and store it in the instance’s `__dict__`.  
Subsequent lookups find the instance attribute first (non‑overriding), so the descriptor is bypassed.  
`functools.cached_property` works the same way.

```python
class Lazy:
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        if instance is None:
            return self
        value = expensive_computation()
        instance.__dict__[self.name] = value   # store in instance
        return value

def expensive_computation():
    print("computing...")
    return 42

class Demo:
    result = Lazy()

d = Demo()
print(d.result)   # computing... 42
print(d.result)   # 42 (no recomputation)
```

---

### Slide 14 – Non‑Special Methods Can Be Shadowed; Special Methods Cannot

Non‑special methods (e.g., `my_method`) are non‑overriding descriptors – an instance attribute of the same name will shadow them.  
Special methods (e.g., `__str__`) are always looked up on the **class**, so instance attributes cannot override them.

```python
class Demo:
    def my_method(self):
        return "original"

d = Demo()
d.my_method = lambda: "shadow"
print(d.my_method())   # "shadow" (instance attribute wins)

# But for special methods:
class Demo2:
    def __str__(self):
        return "original"

d2 = Demo2()
d2.__str__ = lambda: "shadow"
print(str(d2))         # "original" (class method used)
```

---

### Slide 15 – The `__delete__` Method

Implement `__delete__` to handle `del obj.attr` on a managed attribute.

```python
class Managed:
    def __set_name__(self, owner, name):
        self.name = name
    def __get__(self, instance, owner):
        return instance.__dict__.get(self.name)
    def __set__(self, instance, value):
        instance.__dict__[self.name] = value
    def __delete__(self, instance):
        print(f"Deleting {self.name}")
        del instance.__dict__[self.name]

class Demo:
    x = Managed()

d = Demo()
d.x = 5
del d.x   # Deleting x
```

---

### Slide 16 – Descriptors Underpin Major Class Features

Every major class feature in Python is built on descriptors:

- **Instance methods** – function objects with `__get__` returning bound methods
- **Static methods** – `staticmethod` descriptor that returns the plain function
- **Class methods** – `classmethod` descriptor that binds the class as first argument
- **Properties** – `property` descriptor with getter/setter/deleter
- **`__slots__`** – implemented via descriptors that manage a fixed‑size array

```python
class Example:
    def instance_method(self): pass
    @staticmethod
    def static_method(): pass
    @classmethod
    def class_method(cls): pass
    @property
    def prop(self): return 1

# All of these are descriptors attached to the class
print(type(Example.__dict__['instance_method']))  # <class 'function'>
print(type(Example.__dict__['static_method']))    # <class 'staticmethod'>
print(type(Example.__dict__['class_method']))     # <class 'classmethod'>
print(type(Example.__dict__['prop']))             # <class 'property'>
```

---

**That covers every point from the Attribute Descriptors section.** We've seen how descriptors give you fine‑grained control over attribute access, how they relate to properties and methods, and when to use each pattern.
