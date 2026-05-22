# Inheritance: For Better or For Worse

Inheritance is a powerful but easily misused tool. This section of the notes covers the mechanics of `super()`, the pitfalls of subclassing built‑ins, when inheritance makes sense, and when you should reach for composition instead.

---

## 1. How `super` Works

`super(type, object_or_type)` returns a **proxy** that finds the next method in the MRO (Method Resolution Order) *starting after `type`*, and binds it to `object_or_type`.

```python
class Root:
    def describe(self):
        return "Root"

class A(Root):
    def describe(self):
        # super(A, self) looks up 'describe' starting after A in the MRO
        return f"A -> {super().describe()}"

class B(Root):
    def describe(self):
        return f"B -> {super().describe()}"

class C(A, B):
    def describe(self):
        return f"C -> {super().describe()}"

# MRO of C: C -> A -> B -> Root -> object
print(C.__mro__)
# (<class 'C'>, <class 'A'>, <class 'B'>, <class 'Root'>, <class 'object'>)

print(C().describe())
# C -> A -> B -> Root
```

`super()` inside `A.describe` finds `B.describe`, not `Root.describe`—because the MRO is determined by the *runtime class*, which is `C`. This is the dynamic magic of `super`.

---

## 2. Built‑ins Don't Call Overridden Methods

Built‑in types (`dict`, `str`, `list`) are written in C. Their methods often call each other *internally* in C, bypassing any overrides you write in Python.

```python
# ❌ dict.get does NOT call our overridden __getitem__
class LoudDict(dict):
    def __getitem__(self, key):
        print(f"Accessing key: {key}")
        return super().__getitem__(key)

ld = LoudDict(a=1, b=2)
print(ld['a'])      # Accessing key: a → 1  (our method is called)
print(ld.get('b'))  # 2  (NO message! dict.get calls C internals, not our __getitem__)
```

**The fix:** subclass from the user‑friendly versions in `collections` instead.

```python
from collections import UserDict

# ✅ UserDict delegates through Python methods, so overrides work
class LoudUserDict(UserDict):
    def __getitem__(self, key):
        print(f"Accessing key: {key}")
        return super().__getitem__(key)

lud = LoudUserDict(a=1, b=2)
print(lud['a'])      # Accessing key: a → 1
print(lud.get('b'))  # Accessing key: b → 2  (our method IS called!)
```

| Built-in | Use instead |
|---|---|
| `dict` | `collections.UserDict` |
| `list` | `collections.UserList` |
| `str` | `collections.UserString` |

---

## 3. Cooperative Methods and `super`

A method that calls `super()` is a **cooperative method**—it enables cooperative multiple inheritance. Critically, cooperative methods must have **compatible signatures**, because any class in the MRO chain might be called next.

```python
class Animal:
    def __init__(self, name, **kwargs):
        self.name = name
        super().__init__(**kwargs)  # Forward unused kwargs up the chain

class Flyer:
    def __init__(self, max_altitude, **kwargs):
        self.max_altitude = max_altitude
        super().__init__(**kwargs)

class Swimmer:
    def __init__(self, max_depth, **kwargs):
        self.max_depth = max_depth
        super().__init__(**kwargs)

class Duck(Animal, Flyer, Swimmer):
    def __init__(self, name, max_altitude, max_depth):
        super().__init__(name=name, max_altitude=max_altitude, max_depth=max_depth)

    def __repr__(self):
        return f"Duck({self.name!r}, alt={self.max_altitude}, depth={self.max_depth})"

d = Duck("Donald", max_altitude=500, max_depth=10)
print(d)  # Duck('Donald', alt=500, depth=10)
```

Using `**kwargs` lets each class consume the parameters it cares about and forward the rest up the chain. This is the standard pattern for cooperative multiple inheritance.

**Without compatible signatures, cooperative inheritance breaks:**
```python
class BadFlyer:
    def __init__(self, max_altitude):   # No **kwargs!
        self.max_altitude = max_altitude
        super().__init__()              # Calls next in MRO with no arguments

class BadDuck(Animal, BadFlyer):
    def __init__(self, name, max_altitude):
        super().__init__(name=name, max_altitude=max_altitude)

# Duck("Daffy", 300)
# TypeError: __init__() takes 1 positional argument but 2 were given
```

---

## 4. `super` is Dynamic—Even Without a Superclass

A method in a class with no explicit parent can still call `super()` successfully, because the MRO is determined at runtime.

```python
class NoParent:
    def method(self):
        result = super().method()  # Seems like this should fail...
        return f"NoParent -> {result}"

# Standalone: this would fail
# NoParent().method()  # AttributeError: 'super' object has no attribute 'method'

# But in a diamond hierarchy, it works!
class HasMethod:
    def method(self):
        return "HasMethod"

class Middle(NoParent, HasMethod):
    pass

print(Middle.__mro__)
# (Middle, NoParent, HasMethod, object)

print(Middle().method())
# NoParent -> HasMethod   ← super() in NoParent found HasMethod.method!
```

`super()` in `NoParent` resolved to `HasMethod` at runtime because of the MRO. This is fragile, but it demonstrates the dynamic power of `super`.

---

## 5. Mixins Should Appear First in Base Classes

A **mixin** is a class that provides additional behavior to other classes but is not meant to stand on its own. When combining mixins with a base class, list the mixin *first* so its methods can properly override those in the other bases.

```python
class DictMixin:
    """Adds JSON serialization to any Mapping."""
    def to_json(self):
        import json
        return json.dumps(dict(self))

class LoggingMixin:
    """Adds logging to __getitem__."""
    def __getitem__(self, key):
        print(f"Fetching {key!r}")
        return super().__getitem__(key)

# ✅ Mixin first, then the concrete base class
class MyDict(LoggingMixin, DictMixin, dict):
    pass

md = MyDict(a=1, b=2)
print(md['a'])        # Fetching 'a' → 1  (LoggingMixin intercepts __getitem__)
print(md.to_json())   # {"a": 1, "b": 2}   (DictMixin provides to_json)

# ❌ If dict were first, LoggingMixin's __getitem__ might not be called
```

The MRO reads left‑to‑right: `MyDict -> LoggingMixin -> DictMixin -> dict -> object`. The mixin methods take priority over `dict`'s methods.

---

## 6. Favour Composition Over Inheritance

Inheritance creates tight coupling. Composition (holding a reference to another object and delegating to it) is often more flexible and easier to test.

```python
# ❌ Inheritance for code reuse
class CPUIntensiveTask:
    def run(self, data):
        # ... heavy computation ...
        return sum(data)

class LoggedTask(CPUIntensiveTask):
    def run(self, data):
        print(f"Starting task with {len(data)} items")
        result = super().run(data)
        print(f"Finished: {result}")
        return result

# ✅ Composition: wrap and delegate
class LoggedTask:
    def __init__(self, task):
        self._task = task         # Composition: hold a reference

    def run(self, data):
        print(f"Starting task with {len(data)} items")
        result = self._task.run(data)  # Delegation
        print(f"Finished: {result}")
        return result

task = CPUIntensiveTask()
logged = LoggedTask(task)
logged.run([1, 2, 3])
# Starting task with 3 items
# Finished: 6
```

With composition, you can wrap *any* object that has a `run` method—not just subclasses of `CPUIntensiveTask`. It's more reusable and testable.

---

## 7. Interface Inheritance vs. Implementation Inheritance

| Type | Relationship | How to do it |
|---|---|---|
| **Interface inheritance** | is‑a | Subclass an ABC or `typing.Protocol` |
| **Implementation inheritance** | code reuse | Use mixins (but prefer composition) |

```python
from abc import ABC, abstractmethod

# ✅ Interface inheritance: defines WHAT, not HOW
class Drawable(ABC):
    @abstractmethod
    def draw(self, canvas) -> None:
        ...

# ✅ Implementation inheritance via mixin: provides HOW
class BorderMixin:
    def draw_border(self, canvas):
        canvas.draw_rectangle(0, 0, 100, 100)

# Concrete class combines interface + mixin
class Button(Drawable, BorderMixin):
    def draw(self, canvas):
        self.draw_border(canvas)
        canvas.draw_text("OK", 40, 50)

# This is correct: Button is-a Drawable (interface), has BorderMixin (implementation)
print(isinstance(Button(), Drawable))  # True
```

---

## 8. ABCs Should Only Subclass ABCs

A class meant to define an interface should be an explicit `ABC` or `Protocol`. And an ABC should only inherit from other ABCs—not from concrete classes.

```python
from abc import ABC, abstractmethod

# ✅ ABC chain: all abstract
class Vehicle(ABC):
    @abstractmethod
    def move(self):
        ...

class WheeledVehicle(Vehicle):
    @abstractmethod
    def num_wheels(self):
        ...

# ✅ Concrete class at the leaf
class Bicycle(WheeledVehicle):
    def move(self):
        return "Pedaling"
    def num_wheels(self):
        return 2

# ❌ Bad: ABC inheriting from a concrete class
# class Car(SomeConcreteClass, ABC):  # Don't do this!
#     ...
```

Keep your ABC hierarchy pure—only abstract classes in the non‑leaf positions.

---

## 9. Mixins Should Never Be Instantiated; Concrete Classes Shouldn't Inherit Only from Mixins

```python
# ❌ A mixin on its own is meaningless
class SerializerMixin:
    """Adds to_dict() behavior."""
    def to_dict(self):
        return self.__dict__

# ❌ Don't instantiate a mixin alone
# s = SerializerMixin()  # This doesn't make sense

# ❌ Concrete class inheriting ONLY from a mixin
class OnlyMixin(SerializerMixin):
    pass  # No real base behavior—what IS this?

# ✅ Correct: mixin + proper base class
class User(SerializerMixin, object):  # Has a real base (object) and the mixin
    def __init__(self, name, email):
        self.name = name
        self.email = email

u = User("Alice", "alice@example.com")
print(u.to_dict())  # {'name': 'Alice', 'email': 'alice@example.com'}
```

---

## 10. Aggregate Classes

An **aggregate class** is built by inheriting from a combination of ABCs and mixins without adding any new behaviour or structure of its own. It's a convenience for clients.

```python
from abc import ABC, abstractmethod
from collections.abc import MutableMapping, Hashable

# Several mixins with different responsibilities
class JSONSerializable:
    def to_json(self):
        import json
        return json.dumps(dict(self))

class ComparableMixin:
    def __eq__(self, other):
        return dict(self) == dict(other)
    def __ne__(self, other):
        return not (self == other)

# Aggregate class: bundles mixins with a concrete type, adds nothing itself
class FancyDict(JSONSerializable, ComparableMixin, dict):
    """A dict with JSON serialization and value comparison. No new code."""
    pass

fd1 = FancyDict(a=1, b=2)
fd2 = FancyDict(a=1, b=2)
print(fd1 == fd2)        # True (from ComparableMixin)
print(fd1.to_json())     # {"a": 1, "b": 2} (from JSONSerializable)
```

The aggregate class is just a name that bundles capabilities for client convenience.

---

## 11. Only Subclass Classes Designed for Subclassing

```python
from typing import final

# ✅ Clearly designed for subclassing (by name and docstring)
class BaseHTTPHandler:
    """Subclass me to handle specific HTTP methods."""
    def handle_get(self):
        raise NotImplementedError

    def handle_post(self):
        raise NotImplementedError

class MyHandler(BaseHTTPHandler):
    def handle_get(self):
        return "Handling GET"

# ✅ Mark a class as NOT for subclassing
@final
class ImmutableConfiguration:
    """This class is sealed—do not subclass."""
    def __init__(self, **settings):
        self._settings = settings

# class ExtendedConfig(ImmutableConfiguration):  # Type checker rejects this!
#     pass

# ✅ Mark a method as NOT for overriding
class BaseClass:
    @final
    def critical_logic(self):
        """This method must not be overridden."""
        return "core behavior"

    def customizable(self):
        """Override this one if needed."""
        return "default"
```

Use `@typing.final` on classes that shouldn't be subclassed and methods that shouldn't be overridden. It works with mypy, pyright, and many IDEs.

---

## 12. Avoid Subclassing Concrete Classes—All Non‑Leaf Classes Should Be Abstract

```python
# ❌ Bad hierarchy: concrete class in the middle
class Animal:
    def speak(self):
        return "???"

class Dog(Animal):      # Animal is concrete—bad for a non-leaf
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

# ✅ Better: abstract non-leaf classes
from abc import ABC, abstractmethod

class Animal(ABC):                  # Abstract
    @abstractmethod
    def speak(self):
        ...

class Pet(Animal, ABC):            # Still abstract (intermediate node)
    @abstractmethod
    def owner(self):
        ...

class Dog(Pet):                    # Concrete (leaf)
    def speak(self):
        return "Woof!"
    def owner(self):
        return "Human"
```

**The rule of thumb:** if it's not a leaf (i.e., other classes inherit from it), make it abstract. Only the final, instantiable classes should be concrete.

---

## Summary

| Principle | Guidance |
|---|---|
| **`super` mechanics** | Returns a proxy to the next class in the MRO; dynamic, not static |
| **Built‑ins & C** | Use `collections.UserDict/List/String` to ensure overrides are called |
| **Cooperative methods** | `super()` + `**kwargs` for compatible signatures |
| **Dynamic MRO** | `super()` works even in classes with no explicit parent (in diamonds) |
| **Mixin ordering** | Mixin first in the base class tuple |
| **Composition > inheritance** | Wrap and delegate instead of subclassing for code reuse |
| **Interface vs. implementation** | ABC/Protocol for interface; mixins (or composition) for implementation |
| **ABC purity** | ABCs should only inherit from ABCs |
| **Mixin rules** | Never instantiate mixins; concrete classes need a real base |
| **Aggregate classes** | Bundle ABCs + mixins without adding new behaviour |
| **`@typing.final`** | Seal classes and methods against subclassing/overriding |
| **Non‑leaf → abstract** | Every class that has subclasses should be abstract |

Used wisely, inheritance creates elegant, maintainable designs. Used carelessly, it creates a tangled web of tight coupling. Let the MRO, ABCs, and composition guide your decisions.
