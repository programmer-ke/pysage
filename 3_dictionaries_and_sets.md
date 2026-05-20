# Dictionaries and Sets

## Merging Dictionaries

```python
# Using ** unpacking (order matters – later keys override earlier)
merged = {'a': 0, **{'x': 1}, 'y': 2, **{'z': 3, 'x': 4}}
print(merged)  # {'a': 0, 'x': 4, 'y': 2, 'z': 3}

# Using the | operator (Python 3.9+)
d1 = {'a': 1, 'b': 2}
d2 = {'b': 3, 'c': 4}
print(d1 | d2)  # new dict: {'a': 1, 'b': 3, 'c': 4}
```

## In‑place Merging

```python
# |= updates the left dict in place (Python 3.9+)
d1 |= d2
print(d1)  # {'a': 1, 'b': 3, 'c': 4}
```

## Pattern Matching on Dicts

```python
order = {'category': 'ice cream', 'flavour': 'vanilla', 'size': 'large'}

# Partial matches: only specified keys are checked
match order:
    case {'category': 'ice cream', **details}:
        # captures the remaining keys in `details`
        print(f"Ice cream details: {details}")
    case {'category': category}:
        print(f"Other category: {category}")
# Output: Ice cream details: {'flavour': 'vanilla', 'size': 'large'}
```

## Using `collections.abc` for Interfaces

```python
from collections.abc import MutableMapping

# Check if an object adheres to the mapping protocol
def add_default(mapping):
    if isinstance(mapping, MutableMapping):
        mapping.setdefault('key', 'default')
    return mapping
```

## Hashable Objects

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        # Hash must be consistent: equal points produce same hash
        return hash((self.x, self.y))

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1 == p2)        # True
print(hash(p1) == hash(p2))  # True (same process)
# Now they can be used as dict keys
points = {p1: "origin offset"}
print(points[Point(1, 2)])  # "origin offset"
```

## `.setdefault` vs `.get` for Missing Keys

```python
store = {}

# .get only returns a default, doesn't insert it
value = store.get('key', 'default')
print(store)  # {}  – key was not added

# .setdefault inserts the default if the key is missing
value = store.setdefault('key', 'default')
print(store)  # {'key': 'default'}
# Avoids second lookup: it does lookup + insert in one step
```

## `__missing__` for Custom Missing‑Key Handling

```python
class MirrorDict(dict):
    def __missing__(self, key):
        # Called by __getitem__ when key is missing
        return key.upper()   # return a fabricated value

md = MirrorDict(a=1)
print(md['a'])      # 1
print(md['b'])      # 'B'  – __missing__ invoked
# Note: .get() does NOT call __missing__, so careful with behaviour
print(md.get('c'))  # None
# For full control, inherit from collections.UserDict
from collections import UserDict
class BetterMirror(UserDict):
    def __missing__(self, key):
        return key.upper()
bm = BetterMirror(a=1)
print(bm['b'])   # 'B'
```

## `collections.OrderedDict` for Reordering Efficiency

```python
from collections import OrderedDict

# OrderedDict preserves insertion order and has efficient move operations
od = OrderedDict(a=1, b=2, c=3)
od.move_to_end('a')        # move key 'a' to the rightmost position
print(od)  # OrderedDict([('b', 2), ('c', 3), ('a', 1)])
od.move_to_end('c', last=False)  # move to the beginning
print(od)  # OrderedDict([('c', 3), ('b', 2), ('a', 1)])
# Useful for LRU caches
```

Here's a minimal LRU cache built with `OrderedDict`:

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)      # mark as recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)  # refresh
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # evict least recently used
```

**Usage:**
```python
cache = LRUCache(2)
cache.put(1, "one")
cache.put(2, "two")
print(cache.get(1))  # "one"  (makes 1 most recent)
cache.put(3, "three")# evicts 2 (least recent)
print(cache.get(2))  # -1
```


## `collections.ChainMap` – Layers of Mappings

```python
from collections import ChainMap

defaults = {'theme': 'light', 'language': 'en'}
user = {'language': 'fr'}

# ChainMap combines multiple mappings; updates affect only the first one
settings = ChainMap(user, defaults)
print(settings['theme'])    # 'light'  (from defaults)
print(settings['language']) # 'fr'     (from user)
settings['theme'] = 'dark'   # writes into user dict
print(user)                  # {'language': 'fr', 'theme': 'dark'}
print(defaults)              # unchanged
```

## `shelve.Shelf` – Persistent Dict‑like Storage

```python
import shelve

# Shelve stores key/value pairs on disk via pickle
with shelve.open('mydata') as db:
    db['list'] = [1, 2, 3]
    db['settings'] = {'color': 'blue'}

# Later, in another run or the same program
with shelve.open('mydata') as db:
    print(db['list'])      # [1, 2, 3]
```

## Dictionary Views and Set Operations

```python
d1 = {'a': 1, 'b': 2, 'c': 3}
d2 = {'b': 20, 'c': 30, 'd': 40}

# Views are live, read‑only projections
keys1 = d1.keys()  # dict_keys object
keys2 = d2.keys()

# Use set operators to avoid loops
common_keys = keys1 & keys2            # intersection
print(common_keys)                     # {'b', 'c'}

all_keys = keys1 | keys2               # union
print(all_keys)                        # {'a', 'b', 'c', 'd'}

only_in_d1 = keys1 - keys2             # difference
print(only_in_d1)                       # {'a'}

# Compare values or items similarly
matching = d1.items() & d2.items()     # keys with equal values
print(matching)                         # {('c', 3)}  (if c:3 matches)
```

## Literal Set Syntax vs Constructor

```python
# Faster and more readable than set([...])
colors = {'red', 'green', 'blue'}   # uses BUILD_SET bytecode, no list creation

# Disassembly comparison
import dis
dis.dis("set([1,2,3])")
# ... LOAD_NAME set, LOAD_CONST (list), CALL_FUNCTION ...
dis.dis("{1,2,3}")
# ... BUILD_SET ...
```

## Memory: Avoid Instance Attributes Outside `__init__`

```python
class Bad:
    def __init__(self):
        self.x = 1
        # y will be added later → each instance gets its own __dict__ table
obj1 = Bad()
obj1.y = 2  # triggers a new per‑instance hash table

class Good:
    __slots__ = ('x', 'y')   # or define all attributes in __init__
    def __init__(self):
        self.x = 1
        self.y = 2
# Using slots avoids __dict__ entirely → memory savings 10‑20%
```

## Summary: `setdefault` vs `get`

```python
# get: return default, but do not modify the dict
val = d.get('missing', 0)   # safe, but if we wanted to set a default we'd need another line

# setdefault: insert and return the default if key missing
val = d.setdefault('missing', 0)  # one operation instead of two (__contains__ + __setitem__)
```
