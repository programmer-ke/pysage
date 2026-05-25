## 1. Why Use a Pool of Workers?

Starting a thread or a process is **expensive**. Instead of creating one for a single computation and then discarding it, we create a **pool** of workers that can handle many tasks, amortising the startup cost.

```python
from concurrent.futures import ThreadPoolExecutor
import time

def work(n):
    time.sleep(0.1)
    return n * n

# Reuse a pool of 3 threads for 10 tasks
with ThreadPoolExecutor(max_workers=3) as executor:
    results = executor.map(work, range(10))
    print(list(results))   # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

The pool keeps threads alive and reuses them across multiple `work()` calls.

---

## 2. Execution Units

Python provides three kinds of **execution units** – each has its own state and call stack:

- **Processes** – heavyweight, isolated memory space.
- **Threads** – lightweight, share memory within a process.
- **Coroutines** – even lighter, run cooperatively in a single thread.

```python
import os, threading, asyncio

# Process – each has its own PID
print("Process ID:", os.getpid())

# Thread – multiple can exist in one process
def show_thread():
    print("Thread name:", threading.current_thread().name)

t = threading.Thread(target=show_thread)
t.start(); t.join()

# Coroutine – just a function with its own stack
async def show_coro():
    print("Coroutine running")

asyncio.run(show_coro())
```

---

## 3. Inter‑Process Communication

Processes have **separate memory spaces**. To exchange data they must use:

- **Pipes** – unidirectional byte streams.
- **Sockets** – network or Unix domain.
- **Memory‑mapped files** – shared regions of a file.

These carry only **raw bytes** – you must serialise/deserialise data.

```python
from multiprocessing import Process, Pipe

def sender(conn):
    conn.send(b"Hello from process")   # raw bytes
    conn.close()

parent_conn, child_conn = Pipe()
p = Process(target=sender, args=(child_conn,))
p.start()
print(parent_conn.recv())   # b"Hello from process"
p.join()
```

---

## 4. Threads – Shared Memory, Pre‑emptive Multitasking

A **thread** runs inside a process, shares memory with sibling threads, and is scheduled **pre‑emptively** by the OS. The OS can suspend a thread at any point (e.g., every few milliseconds) and switch to another.

```python
import threading, time

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:           # without the lock we'd get race conditions
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)   # 400000 (correct, thanks to the lock)
```

Without the lock, threads would corrupt the shared `counter` because the `+=` operation is not atomic.

---

## 5. Coroutines – Cooperative Multitasking

**Coroutines** are functions that can **suspend** themselves and resume later. They run under an event loop in a single thread and use **cooperative multitasking** – each coroutine voluntarily yields control via `await` or `yield`.

```python
import asyncio

async def say_after(delay, msg):
    await asyncio.sleep(delay)    # suspend here, let others run
    print(msg)

async def main():
    # Two coroutines running concurrently but cooperatively
    task1 = asyncio.create_task(say_after(1, "Hello"))
    task2 = asyncio.create_task(say_after(0.5, "World"))
    await task1
    await task2

asyncio.run(main())
# Output (after ~1 second):
# World
# Hello
```

Classic coroutines (pre‑`async`/`await`) were built from generators using `yield` and `send()`. Native coroutines use `async def` and `await`.

---

## 6. Queues – Exchanging Data Between Execution Units

Queues let threads or coroutines **produce and consume** items safely, acting as a buffer and coordination mechanism.

**Thread queue:**

```python
from queue import Queue
from threading import Thread

q = Queue()

def worker():
    while True:
        item = q.get()
        if item is None:   # poison pill
            break
        print(f"Processing {item}")
        q.task_done()

t = Thread(target=worker, daemon=True)
t.start()
for i in range(5):
    q.put(i)
q.put(None)   # sentinel
t.join()
```

**Coroutine queue (asyncio):**

```python
import asyncio

async def worker(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Coroutine processing {item}")

async def main():
    q = asyncio.Queue()
    task = asyncio.create_task(worker(q))
    for i in range(5):
        await q.put(i)
    await q.put(None)
    await task

asyncio.run(main())
```

---

## 7. Locks – Synchronising Access

A **lock** ensures that only one execution unit can access a critical section at a time.

```python
import threading

shared_list = []
lock = threading.Lock()

def append_safely(value):
    with lock:
        shared_list.append(value)

threads = [threading.Thread(target=append_safely, args=(i,)) for i in range(10)]
for t in threads: t.start()
for t in threads: t.join()
print(len(shared_list))   # 10 – no items lost
```

For coroutines, `asyncio.Lock` provides the same mutual exclusion without blocking the whole thread.

---

## 8. The Global Interpreter Lock (GIL)

The GIL is a mutex that protects access to Python objects, allowing **only one thread** to execute Python bytecode at a time. This serialises execution on multi‑core systems for CPU‑bound code.

**Key facts:**

- The interpreter **releases the GIL every ~5 ms**, giving other threads a chance.
- **C extensions** (e.g., NumPy) and **system calls** (I/O) release the GIL while running.
- Non‑Python threads created by C extensions are **not affected** by the GIL.
- I/O operations release the GIL, so **network programming is mostly unaffected**.

```python
import threading, time

def cpu_bound():
    total = 0
    for i in range(10_000_000):
        total += i
    return total

# Running CPU‑bound work in threads is *slower* than sequentially
# because of GIL contention and context‑switch overhead.
start = time.time()
threads = [threading.Thread(target=cpu_bound) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()
print("Threaded time:", time.time() - start)

start = time.time()
cpu_bound()
cpu_bound()
print("Sequential time:", time.time() - start)
# Sequential is often faster for pure CPU work.
```

---

## 9. Multithreading + Coroutines – Best Practice

Run **all coroutines in one thread** with the event loop, and use **additional threads** for blocking or CPU‑intensive tasks.

```python
import asyncio, time

def blocking_task():
    time.sleep(2)            # blocks the thread, but not the event loop thread
    return "Blocking result"

async def main():
    loop = asyncio.get_running_loop()
    # Run blocking function in a separate thread
    result = await loop.run_in_executor(None, blocking_task)
    print(result)

asyncio.run(main())
```

This keeps the event loop responsive while offloading heavy work.

---

## 10. `time.sleep` – Blocks the Thread but Releases the GIL

Calling `time.sleep(n)` pauses the **calling thread** for `n` seconds, but **releases the GIL**, allowing other threads to run.

```python
import threading, time

def sleepy(name):
    print(f"{name} sleeping...")
    time.sleep(1)            # other threads can run while this one sleeps
    print(f"{name} awake")

threads = [threading.Thread(target=sleepy, args=(f"T{i}",)) for i in range(3)]
for t in threads: t.start()
for t in threads: t.join()
# All three finish in ~1 second, not 3.
```

---

## 11. `threading.Event` – Simple Signalling

An `Event` is a flag that one thread sets and others wait for – the simplest coordination primitive.

```python
import threading

event = threading.Event()

def waiter():
    print("Waiter: waiting for event...")
    event.wait()             # blocks until event is set
    print("Waiter: event received!")

t = threading.Thread(target=waiter)
t.start()
import time; time.sleep(1)
print("Main: setting event")
event.set()
t.join()
```

---

## 12. `multiprocessing.Event` – Cross‑Process Signalling

The `multiprocessing.Event` works the same way but **crosses process boundaries**. It is implemented using a low‑level OS semaphore.

```python
from multiprocessing import Process, Event

def waiter(event):
    print("Child: waiting...")
    event.wait()
    print("Child: event received!")

e = Event()
p = Process(target=waiter, args=(e,))
p.start()
import time; time.sleep(1)
print("Main: setting event")
e.set()
p.join()
```

---

## 13. Pre‑emptive vs. Cooperative Multitasking

- **Threads & Processes**: pre‑emptive – the OS scheduler allocates CPU time slices. Your code can be interrupted at any point.
- **Coroutines**: cooperative – the application‑level event loop switches between coroutines only at `await` points. You are in control of when you yield.

This makes coroutines **easier to reason about** because context switches only happen at well‑defined points.

---

## 14. Three Ways to Run a Coroutine

1. **`asyncio.run(coro())`** – drives a coroutine from synchronous code.
2. **`asyncio.create_task(coro())`** – schedules a coroutine to run concurrently, from within a running event loop.
3. **`await coro()`** – suspends the current coroutine until `coro()` finishes, then returns its result.

```python
import asyncio

async def inner():
    return 42

async def outer():
    # Way 3: await
    result = await inner()
    print(result)

    # Way 2: create_task
    task = asyncio.create_task(inner())
    print(await task)

# Way 1: asyncio.run
asyncio.run(outer())
```

---

## 15. `asyncio.sleep` vs `time.sleep`

- **`time.sleep(n)`** blocks the entire thread – all coroutines freeze.
- **`asyncio.sleep(n)`** suspends only the current coroutine, allowing others to run.

```python
import asyncio, time

async def bad_coro():
    time.sleep(1)          # blocks the whole thread!
    print("Bad")

async def good_coro():
    await asyncio.sleep(1) # only this coroutine pauses
    print("Good")

async def main():
    # Both scheduled concurrently – good_coro prints first
    task1 = asyncio.create_task(bad_coro())
    task2 = asyncio.create_task(good_coro())
    await task1
    await task2

asyncio.run(main())
# Output:
# Good     (after ~1 second)
# Bad      (after ~2 seconds! because time.sleep blocked everything)
```

---

## 16. Cancellation – Easier with Coroutines

Cancelling a thread is messy and unsafe. **Coroutines can be cancelled cleanly** by catching `asyncio.CancelledError` at `await` points.

```python
import asyncio

async def long_running():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("Cancelled – cleaning up...")
        raise    # re‑raise to propagate cancellation

async def main():
    task = asyncio.create_task(long_running())
    await asyncio.sleep(0.1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Main: task was cancelled")

asyncio.run(main())
# Output:
# Cancelled – cleaning up...
# Main: task was cancelled
```

Because coroutines only suspend at `await`, there are predictable places to catch the cancellation and clean up.

---

## 17. The Worker Pattern with a Poison Pill

A common pattern: workers loop forever, pulling tasks from a queue. A special **sentinel value** (a “poison pill”) tells them to stop.

```python
from queue import Queue
from threading import Thread

def worker(q: Queue):
    while True:
        item = q.get()
        if item is None:         # poison pill
            print("Worker shutting down")
            break
        print(f"Processing {item}")

q = Queue()
threads = [Thread(target=worker, args=(q,)) for _ in range(2)]
for t in threads: t.start()

for i in range(5):
    q.put(i)

# Send one poison pill per worker
for _ in threads:
    q.put(None)

for t in threads: t.join()
```

The same pattern works for coroutines with `asyncio.Queue` and `await q.get()`.

---

## Summary

| Concept | Threads | Processes | Coroutines |
|---------|---------|-----------|------------|
| Memory | Shared | Isolated | Shared (one thread) |
| Scheduling | Pre‑emptive (OS) | Pre‑emptive (OS) | Cooperative (event loop) |
| Communication | Shared data + locks | Pipes, sockets, mmap | Async queues, shared state (beware) |
| Cancellation | Difficult | Terminate signal | Clean via `CancelledError` |
| GIL affected | Yes (CPU‑bound) | No | No (single thread) |

- Use **threads** for I/O‑bound work (GIL released).
- Use **processes** for CPU‑bound work (bypass the GIL).
- Use **coroutines** for high‑concurrency network servers with clean, predictable cancellation.
