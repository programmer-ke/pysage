# Object References, Mutability and Recycling

## Variables Are Bindings, Not Assignments

```python
a = [1, 2, 3]   # 'a' is bound to a list object
b = a           # 'b' is bound to the same object
b.append(4)
print(a)        # [1, 2, 3, 4]  — both names refer to the same object
```

---

## Object Identity Never Changes

```python
x = [10, 20]
print(id(x))          # e.g., 140234567890
x.append(30)
print(id(x))          # same id — object mutated, identity unchanged
```

---

## The `is` Operator

```python
# Use 'is' for identity comparison, especially with singletons
a = None
print(a is None)      # True

# 'is' is faster than '==' because it cannot be overloaded
# '==' calls __eq__, 'is' compares memory addresses directly
```

---

## Shallow Copies

```python
import copy

original = [[1, 2], [3, 4]]
shallow1 = original[:]          # slice copy
shallow2 = list(original)       # constructor copy
shallow3 = copy.copy(original)  # copy.copy

shallow1[0].append(99)
print(original)   # [[1, 2, 99], [3, 4]]  — inner list shared
```

---

## Deep Copies

```python
import copy

original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)
deep[0].append(99)
print(original)   # [[1, 2], [3, 4]]  — completely independent

# Handles cyclic references gracefully
a = []
a.append(a)
b = copy.deepcopy(a)  # no infinite recursion
```

---

## Customising Copy Behaviour

```python
import copy

class MyClass:
    def __copy__(self):
        print("Shallow copy customised")
        return MyClass()

    def __deepcopy__(self, memo):
        print("Deep copy customised")
        return MyClass()

obj = MyClass()
shallow = copy.copy(obj)   # prints "Shallow copy customised"
deep = copy.deepcopy(obj)  # prints "Deep copy customised"
```

---

## Call by Sharing

```python
def modify(lst):
    lst.append(4)          # mutates the shared object

nums = [1, 2, 3]
modify(nums)
print(nums)                # [1, 2, 3, 4]  — argument is an alias
```

---

## Avoid Mutating Arguments

```python
# Principle of least astonishment: work on copies inside functions
def safe_append(lst, item):
    new_list = list(lst)   # or lst.copy()
    new_list.append(item)
    return new_list

original = [1, 2]
result = safe_append(original, 3)
print(original)  # [1, 2]  — unchanged
print(result)    # [1, 2, 3]
```

---

## `del` Deletes References, Not Objects

```python
a = [1, 2, 3]
b = a
del a           # removes the name 'a', not the object
print(b)        # [1, 2, 3]  — object still alive via 'b'
```

---

## Garbage Collection via Reference Counting

```python
class Resource:
    def __del__(self):
        print("Resource cleaned up")

r = Resource()
r = None          # reference count drops to 0 → __del__ called
# Output: Resource cleaned up
```

---

## Weak References

```python
import weakref

class Data:
    pass

obj = Data()
weak = weakref.ref(obj)   # does not increase reference count
print(weak())             # <__main__.Data object ...>

del obj
print(weak())             # None  — object garbage collected despite weak ref
```

```python
# Useful for caches: prevent cache from keeping objects alive
cache = weakref.WeakValueDictionary()
key = "user"
cache[key] = Data()
print(cache.get(key))     # <Data object>
del key
# object may be collected; cache entry disappears automatically
```

---

## Interning (Optimisation Detail)

```python
# Small integers and some strings are interned to avoid duplication
a = 256
b = 256
print(a is b)   # True  — same object (CPython implementation detail)

x = 257
y = 257
print(x is y)   # False — not interned (may vary)

s1 = "hello"
s2 = "hello"
print(s1 is s2) # True  — string interning
```
