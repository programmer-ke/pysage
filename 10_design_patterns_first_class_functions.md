# Design Patterns with First-Class Functions

Python’s first-class functions—functions that can be passed as arguments, returned from other functions, and assigned to variables—allow us to simplify several classic object-oriented design patterns. In this presentation we’ll look at the **Strategy** and **Command** patterns, how **decorators** can be used to register strategies, and how **`__call__`** can replace single-method classes.

---

## 1. Simplifying the Strategy Pattern

### Traditional OOP Approach

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. In classic OOP, this often means creating multiple classes:

```python
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, order_total): ...

class NoDiscount(DiscountStrategy):
    def apply(self, order_total):
        return order_total

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent):
        self.percent = percent
    def apply(self, order_total):
        return order_total * (1 - self.percent)

class Order:
    def __init__(self, total, strategy: DiscountStrategy):
        self.total = total
        self.strategy = strategy

    def final_price(self):
        return self.strategy.apply(self.total)

order = Order(100, PercentageDiscount(0.1))
print(order.final_price())  # 90.0
```

### With First-Class Functions

Since functions are objects, we can simply pass a function as the strategy:

```python
def no_discount(total):
    return total

def percentage_discount(percent):
    def apply(total):
        return total * (1 - percent)
    return apply

class Order:
    def __init__(self, total, discount_fn):
        self.total = total
        self.discount_fn = discount_fn

    def final_price(self):
        return self.discount_fn(self.total)

order = Order(100, percentage_discount(0.1))
print(order.final_price())  # 90.0
```

No interface, no abstract base class, no extra classes. The strategy is just a callable.

---

## 2. Simplifying the Command Pattern

### Traditional OOP Approach

The Command pattern encapsulates a request as an object, allowing you to parameterize clients with different requests, queue or log requests, and support undoable operations.

```python
class Command(ABC):
    @abstractmethod
    def execute(self): ...

class SaveCommand(Command):
    def __init__(self, document):
        self.document = document
    def execute(self):
        self.document.save()

class UndoCommand(Command):
    def __init__(self, document):
        self.document = document
    def execute(self):
        self.document.undo()

class Macro:
    def __init__(self):
        self.commands = []
    def add(self, command: Command):
        self.commands.append(command)
    def run(self):
        for cmd in self.commands:
            cmd.execute()
```

### With First-Class Functions

A command is just a callable. We can use plain functions or any object with `__call__`:

```python
class Document:
    def save(self):
        print("Document saved")
    def undo(self):
        print("Undo last action")

def save_cmd(document):
    document.save()

def undo_cmd(document):
    document.undo()

class Macro:
    def __init__(self):
        self.commands = []
    def add(self, command):
        self.commands.append(command)
    def run(self):
        for cmd in self.commands:
            cmd()

doc = Document()
macro = Macro()
macro.add(lambda: save_cmd(doc))
macro.add(lambda: undo_cmd(doc))
macro.run()
# Output:
# Document saved
# Undo last action
```

No need for a formal `Command` interface; anything callable works.

---

## 3. Decorators for Registering Strategy Functions

Decorators can be used to automatically register functions in a strategy registry, removing the need for manual registration.

```python
# Strategy registry
discount_strategies = {}

def register(name):
    """Decorator that registers the function under a given name."""
    def decorator(fn):
        discount_strategies[name] = fn
        return fn
    return decorator

@register("standard")
def standard_discount(total):
    return total * 0.9

@register("vip")
def vip_discount(total):
    return total * 0.8

def apply_discount(name, total):
    strategy = discount_strategies.get(name)
    if strategy:
        return strategy(total)
    raise ValueError(f"No strategy named {name}")

print(apply_discount("vip", 100))   # 80.0
print(apply_discount("standard", 100))  # 90.0
```

The decorator `@register(...)` both creates the function and adds it to the registry. This is a clean, declarative way to build a plug-in system of strategies.

---

## 4. Replacing Single-Method Classes with `__call__`

When a class exists only to provide a single method (e.g., `execute`), we can make it a callable by implementing `__call__`. This allows the object to be used wherever a function is expected, while still retaining internal state.

**Before: a single-method class**
```python
class Greeter:
    def __init__(self, greeting):
        self.greeting = greeting
    def greet(self, name):
        return f"{self.greeting}, {name}!"

g = Greeter("Hello")
print(g.greet("Alice"))  # Hello, Alice!
```

**After: using `__call__`**
```python
class Greeter:
    def __init__(self, greeting):
        self.greeting = greeting
    def __call__(self, name):
        return f"{self.greeting}, {name}!"

g = Greeter("Hello")
print(g("Alice"))  # Hello, Alice!
```

Now `g` is a callable object, which can be passed to any API expecting a function. This is especially useful in the Command pattern, where a command might need internal state (e.g., a counter, a reference to a receiver) but should also be callable like a plain function.

**Example: a stateful command as a callable**
```python
class Counter:
    def __init__(self):
        self.count = 0
    def __call__(self):
        self.count += 1
        print(f"Command executed {self.count} times")

cmd = Counter()
cmd()
cmd()
# Output:
# Command executed 1 times
# Command executed 2 times
```

---

## Summary

| Pattern | Classic OOP | Pythonic Approach |
|---|---|---|
| **Strategy** | Multiple classes with a common interface | Pass a function or callable directly |
| **Command** | Encapsulate request in an object with `execute()` | Use functions / lambdas or callable objects |
| **Strategy registration** | Manual mapping of names to instances | Decorators to build a registry automatically |
| **Single-method class** | Class with one method | Add `__call__` to make the instance a callable |

First-class functions make these patterns less verbose, more flexible, and aligned with Python’s philosophy of “everything is an object.” Use them to write cleaner, more idiomatic code.
