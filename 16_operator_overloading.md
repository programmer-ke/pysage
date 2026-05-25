## 1. Limitations – What You Can’t Do

**Operators for built-in types are sacred.**  
You cannot change the meaning of `+` for `int`, `str`, or any built‑in class.  
You cannot invent new operators (only the existing set can be overloaded).  
Finally, the **logical** operators `is`, `and`, `or`, and `not` are **never** overloadable.

```python
# Trying to change int's '+' would fail – the language simply won't allow it
# (There is no hook for this on built-in types.)
```

**Why?**  
Python’s design adds flexibility *without* breaking the fundamental building blocks.

---

## 2. Unary Operators

Unary operators (`-`, `+`, `~`) are implemented via `__neg__`, `__pos__`, and `__invert__`.  
The rule: **always create and return a new object**, never modify `self` in place.

```python
class Vector:
    def __init__(self, x):
        self.x = x

    def __neg__(self):
        return Vector(-self.x)   # new instance

    def __pos__(self):
        # + is identity; we can return self if object is immutable
        return self               # Vector is immutable, so safe

    def __invert__(self):
        return Vector(~self.x)    # bitwise NOT

v = Vector(5)
print((-v).x)   # -5
print((+v).x)   # 5
print((~v).x)   # -6
```

> **Special case:** If your class is immutable (like `str`), `__pos__` can simply return `self` – no need to copy.

---

## 3. Infix Operators – Return New Objects

For **infix** (`+`, `-`, `*`, etc.) and **unary** operators, return a **fresh** object.  
Only **augmented assignment** (`+=`, `*=`, etc.) may mutate the left operand *if it is mutable*.

```python
class MutableBag:
    def __init__(self, items):
        self.items = list(items)

    def __iadd__(self, item):
        self.items.append(item)   # mutate in place
        return self               # must return self

bag = MutableBag([1,2])
bag += 3
print(bag.items)   # [1, 2, 3]
```

> **Remember:** `a += b` tries `a.__iadd__(b)`; if it doesn’t exist, Python falls back to `a = a + b` (creating a new object).

---

## 4. How Infix Special Methods Dispatch

When Python sees `a + b`, it does:
1. Call `type(a).__add__(a, b)`. If it returns something other than `NotImplemented`, use that.
2. Otherwise, call `type(b).__radd__(b, a)` (the **reflected** version).
3. If that also fails (`NotImplemented`), raise `TypeError`.

The special value `NotImplemented` is a **singleton** – different from `NotImplementedError`.

```python
class Left:
    def __add__(self, other):
        if isinstance(other, Right):
            return "Left + Right"
        return NotImplemented

class Right:
    def __radd__(self, other):
        if isinstance(other, Left):
            return "Right r-add Left"
        return NotImplemented

l = Left()
r = Right()
print(l + r)   # "Left + Right"  (Left.__add__ succeeds)

# If Left didn't know Right, it returns NotImplemented and Right.__radd__ is tried
```

---

## 5. `NotImplemented` vs `NotImplementedError`

- **`NotImplemented`** – a singleton sentinel that tells Python: “I don’t know how to do this; try the reflected method.”
- **`NotImplementedError`** – an exception raised by abstract base classes (ABCs) to indicate unimplemented methods.

```python
class HalfBaked:
    def __add__(self, other):
        if isinstance(other, int):
            return self.value + other
        return NotImplemented   # delegate to other's __radd__
```

Using `NotImplemented` is **essential** for operator interoperability.

---

## 6. Commutative Operations and Delegation

For **commutative** operations like + and *, you can make your forward method delegate to the reverse method when it encounters an unknown type. This avoids duplication.

```python
class Number:
    def __init__(self, n):
        self.n = n

    def __add__(self, other):
        if isinstance(other, Number):
            return Number(self.n + other.n)
        # Delegate to other’s __radd__ (if any)
        return NotImplemented

    def __radd__(self, other):
        return self + other   # now forward __add__ will handle it

a = Number(3)
print(a + 5)      # -> Number(8) – after int.__radd__? Actually, int doesn't know Number...
# Let's test: a + 5 → Number.__add__ sees int, returns NotImplemented, Python tries int.__radd__(5, a)
# int.__radd__ doesn't exist, so TypeError. So delegation only works if other type implements __radd__.
# A better delegation pattern: __add__ can try to coerce or raise, but the example shows the idea.
```

**Better approach**: in `__add__`, after checking known types, you might do `return other.__radd__(self)` as a last resort, but the cleanest way is to return `NotImplemented` and let the language try `other.__radd__` — assuming `other` implements it.

---

## 7. Catching Exceptions in Operators

If an operator method raises an exception, Python **stops** the dispatch immediately. It’s often better to catch anticipated errors and return `NotImplemented` so the reflected method gets a chance.

```python
class MaybeAddable:
    def __add__(self, other):
        try:
            # try something that might fail
            result = self.data + other.data
            return result
        except AttributeError:
            return NotImplemented   # let other handle it
```

Now improper accesses won’t kill the whole operation – they fall back politely.

---

## 8. Rich Comparison Operators

For `==`, `!=`, `<`, `>`, `<=`, `>=`, Python uses the same methods for forward and reverse, but **swaps the operands**.

- `a == b` tries `a.__eq__(b)`, then `b.__eq__(a)`.
- `a > b` tries `a.__gt__(b)`, then `b.__lt__(a)` (notice the method changes).

```python
class Point:
    def __init__(self, x):
        self.x = x

    def __eq__(self, other):
        if isinstance(other, Point):
            return self.x == other.x
        return NotImplemented   # allow other type to implement equality

    # No need to write __ne__ ; Python uses the inverse of __eq__
    
p1 = Point(1)
p2 = Point(1)
p3 = Point(2)
print(p1 == p2)   # True
print(p1 != p3)   # True  (automatically from __eq__)
```

> **Note:** `a != b` works by negating `__eq__` unless you explicitly define `__ne__`.

---

## 9. In the Face of Ambiguity, Refuse the Temptation to Guess

When a comparison is unclear (e.g. when types don’t have an obvious order), it’s better to **raise a `TypeError`** or return `NotImplemented` rather than make an arbitrary decision.

```python
class Weird:
    def __lt__(self, other):
        if isinstance(other, Weird):
            # some logic
            return True
        # No obvious comparison with other types → refuse
        raise TypeError("Cannot compare Weird with %r" % type(other))
```

This follows the **Principle of Least Astonishment**: users won’t be surprised by a silent, possibly wrong result.

---

## 10. Augmented Assignment Fallback

If a class does not implement `__iadd__`, `a += b` is translated into `a = a + b` – using `__add__` instead. For immutable objects this is exactly what you want, so you can skip `__iadd__` entirely.

```python
class ImmutableCounter:
    def __init__(self, n):
        self.n = n

    def __add__(self, other):
        return ImmutableCounter(self.n + other)

c = ImmutableCounter(5)
c += 3   # c = c + 3 → c is now a new ImmutableCounter(8)
print(c.n)   # 8
```

`__iadd__` is only needed when you want to **mutate the left object** for efficiency.

---

## 11. `+=` is More Liberal Than `+`

With `+`, the result type is ambiguous when the operands are of different types.  
With `+=`, the result is **always the type of the left operand**. Therefore you can be more flexible.

```python
class MyListLike:
    def __init__(self, items):
        self.items = list(items)

    def __iadd__(self, other):
        # accept anything iterable, not just the same type
        self.items.extend(other)
        return self

ml = MyListLike([1,2])
ml += (3,4)      # works, even though ml + (3,4) would be weird
print(ml.items)  # [1, 2, 3, 4]
```

This matches how `list` itself works: `list.__iadd__` accepts any iterable, while `list.__add__` only accepts `list`.

---

## 12. `isinstance` Tests Are Common

Operator overloading often requires checking the type of the operand to choose the right action. `isinstance` is your friend.

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        elif isinstance(other, (int, float)):
            return Vector(self.x + other, self.y + other)
        return NotImplemented
```

This pattern helps your objects play nicely with other types that might later implement `__radd__`.

---

## 13. `functools.total_ordering` – Fill in Missing Comparisons

If you define `__eq__` and one ordering method (`__lt__`, `__le__`, `__gt__`, or `__ge__`), you can decorate the class with `@functools.total_ordering` and the rest are generated automatically.

```python
from functools import total_ordering

@total_ordering
class Distance:
    def __init__(self, meters):
        self.m = meters

    def __eq__(self, other):
        if isinstance(other, Distance):
            return self.m == other.m
        return NotImplemented

    def __lt__(self, other):
        if isinstance(other, Distance):
            return self.m < other.m
        return NotImplemented

d1 = Distance(100)
d2 = Distance(200)
print(d1 <= d2)   # True  (generated from __eq__ and __lt__)
print(d1 > d2)    # False
```

This saves code and reduces errors — but be careful: the generated methods rely on the correctness of the ones you provide.

---

## Wrap‑Up

Operator overloading brings your own objects into the Python data model, allowing them to work with built‑in syntax seamlessly. Remember:

- Respect the limitations (no new operators, no altering built‑ins).
- Return new objects from unary/infix operators; mutate only in augmented assignments.
- Use `NotImplemented` to enable a smooth fallback to reflected methods.
- For comparisons, only one `__eq__` is usually enough, and `total_ordering` can fill the rest.

Armed with these rules, you can write Pythonic, intuitive classes that “just work” with the language’s operators.
