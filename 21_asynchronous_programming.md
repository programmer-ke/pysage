### Slide 1 – Starting and Driving the Event Loop

`asyncio.run` starts the event loop and returns only when the loop exits.  
`asyncio.get_running_loop` returns the currently running loop (or raises `RuntimeError` if none is active).

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()   # get the loop from inside a coroutine
    print(f"Loop running: {loop}")
    return 42

# asyncio.run starts the loop, runs main(), and returns its result
result = asyncio.run(main())
print(result)   # 42
# After asyncio.run returns, the loop is closed.
```

---

### Slide 2 – Scheduling Concurrent Tasks

`asyncio.create_task` schedules a coroutine for concurrent execution and returns an `asyncio.Task`.  
The caller does *not* need to `await` the task immediately – it runs in the background.

```python
import asyncio

async def greet(name, delay):
    await asyncio.sleep(delay)
    print(f"Hello, {name}!")

async def main():
    # Schedule two tasks to run concurrently
    task1 = asyncio.create_task(greet("Alice", 1))
    task2 = asyncio.create_task(greet("Bob", 0.5))

    # The main coroutine continues without waiting immediately
    print("Tasks scheduled...")
    # ...do other work...

    # Eventually await them to get results (or ensure they finish)
    await task1
    await task2

asyncio.run(main())
# Output:
# Tasks scheduled...
# Hello, Bob!       (finishes first, shorter delay)
# Hello, Alice!
```

---

### Slide 3 – Gathering Results from Multiple Awaitables

`asyncio.gather` accepts one or more awaitables and returns a list of their results after **all** of them complete.

```python
import asyncio

async def fetch(n):
    await asyncio.sleep(0.1 * n)
    return f"Data-{n}"

async def main():
    results = await asyncio.gather(
        fetch(3),
        fetch(1),
        fetch(2),
    )
    print(results)   # ['Data-3', 'Data-1', 'Data-2']

asyncio.run(main())
```

Results are returned in the **same order** the awaitables were passed to `gather`, even if they complete in a different order.

---

### Slide 4 – Getting Results in Completion Order

`asyncio.as_completed` is a generator that yields coroutines which return results **in the order they finish**, not the order they were submitted.

```python
import asyncio

async def fetch(n):
    await asyncio.sleep(0.1 * n)
    return f"Data-{n}"

async def main():
    tasks = [fetch(3), fetch(1), fetch(2)]

    for coro in asyncio.as_completed(tasks):
        result = await coro          # this returns whichever finishes next
        print(result)

asyncio.run(main())
# Output:
# Data-1         (finished first)
# Data-2
# Data-3
```

---

### Slide 5 – `asyncio.gather` vs Asynchronous Comprehensions

An alternative to `gather` is using an async comprehension (`async for` or `await` inside a list comprehension), but `gather` gives **more control over error handling** – a single exception can be caught for the whole group rather than per‑iteration.

```python
import asyncio

async def safe_fetch(n):
    if n == 2:
        raise ValueError("bad data")
    return f"Ok-{n}"

async def main():
    # gather: one exception raised after all finish, accessible via gather
    try:
        results = await asyncio.gather(safe_fetch(1), safe_fetch(2), safe_fetch(3),
                                       return_exceptions=True)
        print(results)   # ['Ok-1', ValueError('bad data'), 'Ok-3']
    except Exception as e:
        print("Caught:", e)

asyncio.run(main())
```

Using `return_exceptions=True`, exceptions become values in the result list rather than being raised. This is harder to replicate with a raw async comprehension.

---

### Slide 6 – Delegating Work to Threads

`asyncio.to_thread` offloads a blocking synchronous call to a thread pool managed by asyncio, keeping the event loop free.

```python
import asyncio
import time

def blocking_work(n):
    time.sleep(1)           # blocks the *thread*, not the event loop
    return n * 2

async def main():
    result = await asyncio.to_thread(blocking_work, 21)
    print(result)            # 42

asyncio.run(main())
```

Without `to_thread`, calling `time.sleep` directly inside a coroutine would freeze the entire event loop.

---

### Slide 7 – `loop.run_in_executor` (Older Alternative)

Before `asyncio.to_thread` (Python 3.9+), `loop.run_in_executor` was the way to delegate work. You can pass an optional `Executor` – default is a thread pool, but you can also use a process pool.

```python
import asyncio
import concurrent.futures

def cpu_heavy(x):
    return sum(i * i for i in range(x))

async def main():
    loop = asyncio.get_running_loop()

    # Use a process pool for CPU-bound work
    with concurrent.futures.ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10_000)
        print(result)

asyncio.run(main())
```

**Important caveat:** `run_in_executor` is **uncancellable** – cancelling the wrapping task does not stop the work already submitted, which can prevent `asyncio.run` from shutting down cleanly.

---

### Slide 8 – Limiting Concurrency with Semaphores

`asyncio.Semaphore` has an internal counter.  
- `acquire()` (awaited) decrements the counter; it **suspends the coroutine** when the counter reaches zero.  
- `release()` increments it, allowing a waiting coroutine to proceed.  
Using it as a **context manager** is safer – it guarantees release even on exceptions.

```python
import asyncio

sem = asyncio.Semaphore(2)          # only 2 tasks at a time

async def limited_task(n):
    async with sem:                  # acquire & release automatically
        print(f"Task {n} started")
        await asyncio.sleep(1)
        print(f"Task {n} done")

async def main():
    await asyncio.gather(*(limited_task(i) for i in range(5)))

asyncio.run(main())
# Output will show at most 2 tasks starting at a time,
# others wait until a slot is released.
```

---

### Slide 9 – Asynchronous Context Managers: `async with`

`async with` works with objects that implement `__aenter__` and `__aexit__` as **coroutines**.  
These are used when entering/exiting a context involves I/O (e.g. acquiring a database connection, opening a network socket).

```python
import asyncio

class AsyncConnection:
    async def __aenter__(self):
        print("Opening connection...")
        await asyncio.sleep(0.1)     # simulate async setup
        return self

    async def __aexit__(self, *exc):
        print("Closing connection...")
        await asyncio.sleep(0.1)     # simulate async teardown

    async def query(self, sql):
        return f"result of '{sql}'"

async def main():
    async with AsyncConnection() as conn:
        res = await conn.query("SELECT 1")
        print(res)

asyncio.run(main())
# Opening connection...
# result of 'SELECT 1'
# Closing connection...
```

---

### Slide 10 – `@asynccontextmanager`

The `@asynccontextmanager` decorator wraps an **async generator** to create an async context manager.  
Code before the `yield` runs on entry; code after the `yield` runs on exit.

```python
import asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_resource(name):
    print(f"Acquiring {name}...")
    await asyncio.sleep(0.1)         # async setup
    try:
        yield f"resource-{name}"     # handed to the `as` variable
    finally:
        print(f"Releasing {name}...")
        await asyncio.sleep(0.1)     # async cleanup

async def main():
    async with managed_resource("db") as res:
        print(f"Using {res}")

asyncio.run(main())
```

---

### Slide 11 – Asynchronous Iterables and `async for`

`async for` works with asynchronous iterables that implement `__aiter__`.  
`__aiter__` should be a **regular** (non‑async) function that returns an **asynchronous iterator** (which has `__anext__` as a coroutine).

```python
import asyncio

class AsyncRange:
    def __init__(self, start, end):
        self.current = start
        self.end = end

    def __aiter__(self):            # regular function, not async
        return self

    async def __anext__(self):      # coroutine
        if self.current >= self.end:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)    # simulate async fetch
        value = self.current
        self.current += 1
        return value

async def main():
    async for n in AsyncRange(0, 5):
        print(n)

asyncio.run(main())
```

---

### Slide 12 – Asynchronous Generator Expressions

You can create an asynchronous generator expression using `async for` to consume an iterable.  
The resulting object is an **async generator**, which you iterate over with `async for`.

```python
import asyncio

async def numbers():
    for i in range(5):
        await asyncio.sleep(0.1)
        yield i

async def main():
    # async generator expression (requires Python 3.6+ syntax inside async def)
    doubled = (n * 2 async for n in numbers())    # Note: 'async for' inside comprehension

    async for value in doubled:
        print(value)

asyncio.run(main())
# Output: 0, 2, 4, 6, 8
```

---

### Slide 13 – No Asynchronous Filesystem API

At this point, `asyncio` does **not** provide a built‑in asynchronous filesystem API.  
Disk I/O remains blocking at the OS level, so reading/writing files in a coroutine still blocks the event loop.  
Workarounds include:

- `asyncio.to_thread` / `loop.run_in_executor` with a thread pool
- Third‑party libraries like `aiofiles`

```python
import asyncio

# Using run_in_executor as a workaround for file I/O
async def read_file(path):
    loop = asyncio.get_running_loop()

    def _read():
        with open(path) as f:
            return f.read()

    return await loop.run_in_executor(None, _read)   # None = default thread pool
```

---

### Slide 14 – `await` Borrows from `yield from`

The `await` keyword reuses the same underlying mechanism as `yield from`.  
Both allow one generator/coroutine to delegate to a sub‑generator/sub‑coroutine, suspending the outer one until the inner one completes.

```python
# Classic generator delegation with yield from
def sub_gen():
    yield 1
    yield 2

def main_gen():
    yield from sub_gen()
    yield 3

print(list(main_gen()))   # [1, 2, 3]


# Coroutine delegation with await (same concept)
async def sub_coro():
    await asyncio.sleep(0.1)
    return 10

async def main_coro():
    result = await sub_coro()    # conceptually similar to yield from
    return result + 1

print(asyncio.run(main_coro()))  # 11
```

---

### Slide 15 – Low‑Level Awaitables

At the lowest level, an awaitable is:
- an object with an `__await__` method that returns an **iterator**, or
- an object written against the Python/C API with a `tp_as_async.am_await` function.

You rarely need to write these directly, but they show how `async`/`await` composes with the iterator protocol.

```python
class SimpleAwaitable:
    def __await__(self):
        # __await__ must return an iterator
        yield            # suspend here
        return 42        # this becomes the value of 'await'

async def main():
    obj = SimpleAwaitable()
    result = await obj
    print(result)        # 42

asyncio.run(main())
```

---

### Slide 16 – `async`/`await` Are Not Tied to a Specific Event Loop

Python's `async`/`await` are language constructs that work with **any** compatible event loop.  
Alternative frameworks like `curio` and `trio` use the same `async def` / `await` syntax but provide their own event loops with different design philosophies.

```python
# Conceptual example – trio uses the same syntax with its own loop
# import trio
#
# async def child():
#     await trio.sleep(1)
#     print("Hello from trio")
#
# trio.run(child)
```

The language does not mandate `asyncio` – it only requires an event loop that understands the awaitable protocol (`__await__`).

---

**Wrap‑up summary:**

| Feature | Purpose |
|---------|---------|
| `asyncio.run` | Start the event loop |
| `create_task` | Schedule concurrent coroutines |
| `gather` | Wait for all, ordered results |
| `as_completed` | Yield results by completion time |
| `to_thread` / `run_in_executor` | Offload blocking work |
| `Semaphore` | Limit concurrency |
| `async with` | Async context managers |
| `async for` | Async iteration |
| `await` ≈ `yield from` | Delegation mechanism |
| Framework‑agnostic | `curio`, `trio`, etc. |
