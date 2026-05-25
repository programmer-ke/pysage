## Part 1: Context Managers – the `with` Statement

### 1. The Class‑Based Protocol: `__enter__` and `__exit__`

A context manager object is any object that implements two dunder methods:

- `__enter__(self)` – called when entering the `with` block.
- `__exit__(self, exc_type, exc_val, exc_tb)` – called when leaving the block.

```python
class ManagedFile:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode

    def __enter__(self):
        self.file = open(self.filename, self.mode)
        print("File opened")
        return self.file       # this object is bound to the 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        print("File closed")
        # Returning None (falsy) means: propagate any exception that occurred

with ManagedFile("test.txt", "w") as f:
    f.write("Hello, context managers!")
# Output:
# File opened
# File closed
```

The `__exit__` method always runs – even if an exception occurs inside the block.

---

### 2. `__enter__` Can Return Any Object

`__enter__` doesn't have to return `self`. It can return anything you want bound to the `as` variable:

```python
class DatabaseConnection:
    def __init__(self, db_url):
        self.db_url = db_url

    def __enter__(self):
        print(f"Connecting to {self.db_url}...")
        # Return a lightweight cursor object, not the whole connection
        return {"cursor": "fake_cursor", "db": self.db_url}

    def __exit__(self, *args):
        print("Disconnecting...")

with DatabaseConnection("postgres://...") as conn:
    print(f"Working with {conn['cursor']}")
# Output:
# Connecting to postgres://...
# Working with fake_cursor
# Disconnecting...
```

This lets you expose a clean, limited interface to the `with` block.

---

### 3. `__exit__` and Exception Propagation

`__exit__` receives three exception arguments. If it returns a **falsy** value (like `None` or `False`), any exception that occurred in the block is **re‑raised** after `__exit__` finishes. Returning a **truthy** value **suppresses** the exception.

```python
class SuppressKeyError:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is KeyError:
            print(f"Suppressed KeyError: {exc_val}")
            return True   # truthy → exception is swallowed
        return False      # falsy → exception propagates

with SuppressKeyError():
    d = {}
    print(d["missing"])   # would raise KeyError
print("Execution continues!")

# Output:
# Suppressed KeyError: 'missing'
# Execution continues!
```

Be careful with this power – silencing exceptions can hide real bugs.

---

### 4. Managing Multiple Context Managers

You can use **parentheses** to manage several context managers in one `with` statement:

```python
with (
    open("input.txt", "r") as f_in,
    open("output.txt", "w") as f_out,
):
    data = f_in.read()
    f_out.write(data.upper())
```

Before Python 3.10 you would write them comma‑separated on one line or nest them. The parenthesised form (PEP 617 / Python 3.10+) is cleaner and more readable.

---

### 5. The `@contextmanager` Decorator: Generator‑Based Context Managers

The `@contextmanager` decorator from `contextlib` wraps a **generator function** that yields exactly once:

- **Before `yield`** – runs when entering the `with` block.
- **After `yield`** – runs when exiting the block.

```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename, mode):
    print("Opening file...")
    f = open(filename, mode)
    try:
        yield f                # hand control to the with block
    finally:
        f.close()
        print("File closed.")

with managed_file("test.txt", "w") as f:
    f.write("Using @contextmanager!")
# Output:
# Opening file...
# File closed.
```

The `try .. finally` ensures the file is closed even if an exception occurs.

---

### 6. Exception Handling in `@contextmanager`

If an exception occurs inside the `with` block, it is **thrown into the generator** at the `yield` line. You can catch it with a `try .. except .. finally`:

```python
@contextmanager
def error_handler():
    print("Entering")
    try:
        yield
    except ValueError as e:
        print(f"Caught ValueError: {e}")
        # Exception is swallowed because we don't re-raise
    finally:
        print("Cleaning up (always runs)")

with error_handler():
    print("Inside block")
    raise ValueError("Something went wrong!")
print("Execution continues.")

# Output:
# Entering
# Inside block
# Caught ValueError: Something went wrong!
# Cleaning up (always runs)
# Execution continues.
```

If **no** exception occurs, Python calls `next()` on the generator to resume execution after `yield`:

```python
@contextmanager
def tracker():
    print("Start")
    yield "resource"
    print("End")   # this runs if no exception

with tracker() as r:
    print(f"Got {r}")
# Output:
# Start
# Got resource
# End
```

---

### 7. `@contextmanager` Generators as Decorators

Generators decorated with `@contextmanager` can themselves be **used as decorators**. This creates a context manager that automatically wraps the decorated function:

```python
from contextlib import contextmanager

@contextmanager
def log_execution(name):
    print(f"[{name}] Starting...")
    yield
    print(f"[{name}] Done.")

# Use as a decorator:
@log_execution("my_task")
def do_work():
    print("Working hard...")
    # The function body runs inside the context manager

do_work()
# Output:
# [my_task] Starting...
# Working hard...
# [my_task] Done.
```

This pattern is useful for logging, timing, or setting up/tearing down resources around function calls.

---

## Part 2: `else` Blocks – Think of It as "Then"

The `else` clause on `for`, `while`, and `try` is unique to Python. The notes suggest the mental substitution **"then"** to understand its meaning.

### 8. `else` on a `for` Loop

The `else` block runs if the loop **completed without a `break`**:

```python
# Example: Search for an item
items = [1, 3, 5, 7]

for item in items:
    if item == 4:
        print("Found it!")
        break
else:
    print("Item not found.")   # runs because no break occurred

# Output: Item not found.

# Compare with:
for item in items:
    if item == 5:
        print("Found it!")
        break
else:
    print("Item not found.")
# Output: Found it!   (else block skipped)
```

This is cleaner than using a `found` flag variable.

---

### 9. `else` on a `while` Loop

The `else` block runs if the loop's exit condition **became falsy** (i.e., the loop ended naturally, without a `break`):

```python
# Drain a queue with a timeout (simulated)
count = 5
while count > 0:
    print(f"Processing, remaining: {count}")
    count -= 1
    if count == 2:
        print("Aborting early!")
        break
else:
    print("All items processed normally.")

# Output:
# Processing, remaining: 5
# Processing, remaining: 4
# Processing, remaining: 3
# Processing, remaining: 2
# Aborting early!
# (else block skipped because of break)

# Without break:
count = 3
while count > 0:
    print(f"Tick: {count}")
    count -= 1
else:
    print("Countdown finished!")
# Output:
# Tick: 3
# Tick: 2
# Tick: 1
# Countdown finished!
```

---

### 10. `else` on a `try` Block

The `else` block runs if **no exception** was raised in the `try` block. This is different from `finally`, which always runs:

```python
def divide(a, b):
    try:
        result = a / b
    except ZeroDivisionError:
        print("Cannot divide by zero!")
    else:
        print(f"Division successful: {result}")
    finally:
        print("Operation attempted.")   # always runs

divide(10, 2)
# Output:
# Division successful: 5.0
# Operation attempted.

divide(10, 0)
# Output:
# Cannot divide by zero!
# Operation attempted.
```

**Why use `else` instead of putting code in the `try` block?** Because code in `else` is **not covered by the `except` clause**. This prevents accidentally catching exceptions from code that shouldn't be guarded:

```python
# BAD: db.commit() is inside try – if commit raises DatabaseError, 
# we'd mistakenly think it's a validation error
try:
    validate(data)
    db.save(data)
    db.commit()
except ValidationError:
    print("Invalid data!")

# GOOD: db.commit() is in else – only validation errors are caught
try:
    validate(data)
    db.save(data)
except ValidationError:
    print("Invalid data!")
else:
    db.commit()   # errors here are not caught by the except above
```

---

## Summary

**`with` blocks** give you deterministic resource management:
- Class‑based: implement `__enter__` and `__exit__`.
- Generator‑based: use `@contextmanager` with a `yield`.
- Both ensure cleanup happens even when exceptions occur.

**`else` blocks** provide a clear way to express "and then, if everything went well":
- On `for`/`while`: runs when no `break` occurred.
- On `try`: runs when no exception was raised (and keeps that code separate from the guarded block).
