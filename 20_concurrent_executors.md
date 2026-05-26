### Slide 1 – Executors Are Context Managers

`ThreadPoolExecutor` and `ProcessPoolExecutor` can be used as context managers.  
When the `with` block ends, `executor.shutdown(wait=True)` is called automatically, waiting for all submitted tasks to finish.

```python
import concurrent.futures
import time

def work(n):
    time.sleep(0.1)
    return n * 2

with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    # Submit several tasks
    future1 = executor.submit(work, 5)
    future2 = executor.submit(work, 7)

# After the block, executor waits for all pending futures to complete.
# No need to call shutdown() explicitly.
```

---

### Slide 2 – `executor.map` Yields Results in Order

`executor.map` returns a generator that produces results in the same order as the input iterable.  
Even if a later task finishes earlier, the generator will yield results in the original order.  
If the first task is slow, the generator blocks until it is ready, even if later tasks have already completed.

```python
args = [3, 1, 2, 4]          # Notice: second argument '1' is the fastest

with concurrent.futures.ThreadPoolExecutor() as executor:
    results = executor.map(work, args)        # generator

    for res in results:
        print(res)           # prints 6, 2, 4, 8  (order preserved)
```

---

### Slide 3 – Futures: Deferred Computations

A `Future` represents a computation that may or may not have finished yet – similar to a JavaScript `Promise`.  
Futures are created *by the executor*; application code should never instantiate them directly.  
The `done()` method checks completion without blocking.

```python
with concurrent.futures.ThreadPoolExecutor() as executor:
    f = executor.submit(work, 10)          # returns a Future

    print(f.done())                        # False (most likely)
    time.sleep(0.2)
    print(f.done())                        # True now
    print("Result:", f.result())           # 20
```

---

### Slide 4 – Callbacks on Futures

Futures support a callback mechanism via `add_done_callback`.  
The callback is executed in the **same thread (or process) that executed the future**, so it is called synchronously when the future completes.

```python
def print_result(future):
    print(f"Got result: {future.result()}")

with concurrent.futures.ThreadPoolExecutor() as executor:
    f = executor.submit(work, 3)
    f.add_done_callback(print_result)
    # When future finishes, print_result is called (in the worker thread)
```

---

### Slide 5 – Getting the Result (Blocking vs. Await)

- For a `concurrent.futures.Future`, `result()` **blocks** the calling thread until the computation is complete (or re‑raises any exception that occurred).
- For an `asyncio.Future`, you **await** the result inside an async function – it suspends the coroutine, not the thread.

```python
# concurrent.futures.Future (blocking)
f = executor.submit(work, 8)
answer = f.result()          # may block if not yet done

# asyncio.Future (non‑blocking for the event loop)
import asyncio
async def main():
    loop = asyncio.get_running_loop()
    fut = loop.create_future()
    # ... (set the future result elsewhere)
    answer = await fut       # suspends coroutine until done
```

---

### Slide 6 – `executor.submit` and `as_completed`

`executor.submit` returns individual `Future` objects.  
`concurrent.futures.as_completed` takes an iterable of futures and returns an iterator that yields them **as they complete**, not in submission order.

This is useful when you want to process results as soon as they are ready, without waiting for an earlier, slower task.

```python
def task(n):
    time.sleep(n * 0.1)      # longer delay for larger n
    return n * 2

with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = [executor.submit(task, i) for i in range(5, 0, -1)]   # 5..1

    for future in concurrent.futures.as_completed(futures):
        # The smallest argument finishes first; we get it immediately
        print(future.result())
    # Output: 2, 4, 6, 8, 10  (ordered by completion, i.e., 1,2,3,4,5)
```

Compare this to `executor.map`, which would have blocked until the task with `n=5` finished, even though tasks with smaller `n` were already done.

---

**Wrap‑up:**  

- Use context manager for clean shutdown.  
- `map` gives ordered results; `as_completed` gives them in completion order.  
- Futures are handles for deferred work – use `done()`, `result()`, and callbacks.  
- Let the framework create futures; never instantiate them yourself.
