# Async & Concurrency in Python

> Deep dive into Threading, Multiprocessing, Asyncio, the GIL, and concurrent design patterns. Crucial for backend and backend-infrastructure interviews.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is the GIL (Global Interpreter Lock)?

**Answer:**

The GIL is a mutex (lock) used by CPython that allows **only one thread to execute Python bytecode at a time**, even on multi-core processors.

**Why does it exist?**
CPython's memory management (reference counting) is not thread-safe. The GIL prevents race conditions where multiple threads could simultaneously increment/decrement a reference count, leading to memory leaks or premature deallocation.

**Impact:**
- **CPU-bound tasks:** Multithreading is useless (or even slower due to context switching overhead) because only one thread runs at a time.
- **I/O-bound tasks:** Multithreading works well! The GIL is released when a thread waits for I/O (network, file, sleep), allowing other threads to run.

*(Note: Python 3.13 introduces an experimental free-threaded (no-GIL) mode.)*

---

### Q2: Compare Threading, Multiprocessing, and Asyncio. When to use which?

**Answer:**

| Feature | `threading` | `multiprocessing` | `asyncio` |
|---------|-------------|-------------------|-----------|
| Concurrency model | Preemptive (OS handles switching) | True Parallelism (multiple processes) | Cooperative (event loop switches at `await`) |
| GIL Impact | Blocked by GIL (runs on 1 core) | Bypasses GIL (runs on N cores) | Bypasses GIL issue (single thread) |
| Memory Overhead | Medium (OS threads) | High (Separate memory spaces) | Very Low (Coroutines) |
| Best For | Fast I/O, legacy code, blocking APIs | CPU-bound tasks (math, data processing) | Massive I/O (10,000+ network connections) |

**Rule of Thumb:**
- **CPU-bound?** Use `multiprocessing`.
- **I/O-bound (fast/few)?** Use `threading`.
- **I/O-bound (slow/many)?** Use `asyncio`.

---

### Q3: How do you create and join a thread?

**Answer:**

```python
import threading
import time

def worker(name):
    print(f"Thread {name} starting")
    time.sleep(1)  # Simulates I/O. GIL is released here!
    print(f"Thread {name} finishing")

# Create threads
t1 = threading.Thread(target=worker, args=("A",))
t2 = threading.Thread(target=worker, args=("B",))

# Start threads
t1.start()
t2.start()

# Wait for threads to complete (join)
t1.join()
t2.join()
print("All done")
```

---

### Q4: Explain `async` and `await` keywords.

**Answer:**

Introduced in Python 3.5, they are the foundation of Python's asynchronous programming.

- `async def`: Defines a **coroutine function**. Calling it doesn't execute the function; it returns a **coroutine object**.
- `await`: Pauses the execution of the current coroutine until the awaited task completes. During this pause, control is returned to the **Event Loop**, which can run other coroutines.

```python
import asyncio

async def fetch_data():
    print("Fetching...")
    await asyncio.sleep(1)  # Non-blocking sleep
    print("Done fetching")
    return {"data": 1}

async def main():
    # Calling fetch_data() returns a coroutine. Await it to run.
    result = await fetch_data()
    print(result)

# Entry point starts the event loop
asyncio.run(main())
```

---

## 🟡 Medium (Intermediate)

### Q5: What is a Race Condition? How do you prevent it in Python?

**Answer:**

A race condition occurs when multiple threads access and modify shared data simultaneously, leading to unpredictable results. Even with the GIL, complex operations (like `x += 1` which is read, add, write) can be interrupted.

**Prevention using a Lock (Mutex):**

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        # acquire() and release() ensure only one thread executes this block
        with lock:  # Context manager automatically handles acquire/release
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Counter: {counter}")  # Will be exactly 1,000,000
```

---

### Q6: How do you share data between Processes?

**Answer:**

Because processes have independent memory spaces, you cannot share variables directly. You must use IPC (Inter-Process Communication) primitives provided by the `multiprocessing` module.

**1. Queue (Thread/Process safe FIFO):**
```python
from multiprocessing import Process, Queue

def producer(q):
    q.put("Data from producer")

def consumer(q):
    print("Received:", q.get())

if __name__ == "__main__":
    q = Queue()
    p1 = Process(target=producer, args=(q,))
    p2 = Process(target=consumer, args=(q,))
    p1.start(); p2.start()
    p1.join(); p2.join()
```

**2. Shared Memory (Value/Array):**
```python
from multiprocessing import Process, Value

def increment(shared_val):
    with shared_val.get_lock():  # Locks are required even for shared memory!
        shared_val.value += 1

if __name__ == "__main__":
    v = Value('i', 0)  # 'i' = integer, 0 = initial value
    processes = [Process(target=increment, args=(v,)) for _ in range(10)]
    for p in processes: p.start()
    for p in processes: p.join()
```

---

### Q7: Explain the `concurrent.futures` module (ThreadPoolExecutor / ProcessPoolExecutor).

**Answer:**

It provides a high-level, modern interface for asynchronously executing callables, replacing the need to manually manage threads or processes.

```python
import concurrent.futures
import time

def process_item(item):
    time.sleep(0.5)
    return item * 2

items = [1, 2, 3, 4, 5]

# Using ThreadPoolExecutor (for I/O bound)
with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    # 1. Using map (preserves order)
    results = list(executor.map(process_item, items))
    print(results)  # [2, 4, 6, 8, 10]
    
    # 2. Using submit (returns Futures, good for tracking completion order)
    futures = [executor.submit(process_item, i) for i in items]
    for future in concurrent.futures.as_completed(futures):
        print(future.result())  # Results print as they finish
```

---

### Q8: How do you run concurrent tasks in `asyncio`? (`gather` vs `TaskGroup`)

**Answer:**

**Python 3.10 and earlier: `asyncio.gather`**
```python
import asyncio

async def fetch(id):
    await asyncio.sleep(1)
    return f"Data {id}"

async def main():
    # Runs all concurrently, returns list of results in order
    results = await asyncio.gather(fetch(1), fetch(2), fetch(3))
    print(results)
```

**Python 3.11+: `asyncio.TaskGroup` (Modern approach)**
TaskGroups provide structural concurrency. If one task fails, the TaskGroup cancels all other running tasks, preventing dangling coroutines (leaks).

```python
async def main_modern():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch(1))
        task2 = tg.create_task(fetch(2))
    
    # Block exits only when all tasks are complete
    print(task1.result(), task2.result())
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q9: How do you combine `asyncio` with blocking synchronous code (like `requests` or CPU-heavy math)?

**Answer:**

You cannot run blocking code directly in an `async` function, or you will freeze the entire Event Loop (stopping all other coroutines). You must offload the blocking code to a thread pool.

```python
import asyncio
import time
import requests

def blocking_io():
    # This blocks! Do not run directly in event loop
    response = requests.get("https://httpbin.org/delay/2")
    return response.status_code

def cpu_heavy():
    # This blocks! Do not run directly in event loop
    return sum(i * i for i in range(10**7))

async def main():
    loop = asyncio.get_running_loop()
    
    # 1. Offload I/O to default ThreadPoolExecutor
    print("Starting I/O...")
    io_result = await loop.run_in_executor(None, blocking_io)
    
    # 2. Offload CPU to a ProcessPoolExecutor
    import concurrent.futures
    with concurrent.futures.ProcessPoolExecutor() as pool:
        print("Starting CPU task...")
        cpu_result = await loop.run_in_executor(pool, cpu_heavy)
    
    # Python 3.9+ simplified method for Threads:
    # io_result = await asyncio.to_thread(blocking_io)

asyncio.run(main())
```

---

### Q10: Explain Deadlocks and Livelocks. How do you prevent deadlocks?

**Answer:**

- **Deadlock:** Thread A holds Lock 1 and waits for Lock 2. Thread B holds Lock 2 and waits for Lock 1. Both wait forever.
- **Livelock:** Threads constantly change state in response to each other without making progress (like two people trying to pass each other in a hallway, both stepping left, then right, repeatedly).

**Deadlock Example:**
```python
import threading
lock1 = threading.Lock()
lock2 = threading.Lock()

def func_a():
    with lock1:
        with lock2:  # Danger!
            pass

def func_b():
    with lock2:
        with lock1:  # Danger!
            pass
```

**Deadlock Prevention Strategies:**
1. **Lock Ordering:** Always acquire locks in the exact same predefined order across all threads. (If both functions acquired `lock1` then `lock2`, deadlock is impossible).
2. **Timeouts:** Use `lock.acquire(timeout=2)`. If it fails, release held locks, wait randomly, and retry.
3. **Lock Hierarchies / Context Managers:** Create a higher-level context manager that acquires multiple locks atomically in sorted order.

---

### Q11: Implement a thread-safe Singleton using double-checked locking.

**Answer:**

Standard singletons can fail under heavy concurrency if two threads check `if not _instance` simultaneously.

```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        # 1st check: Avoid locking overhead if already initialized
        if cls._instance is None:
            # 2nd check: Inside the lock, ensure only one thread creates it
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

---

### Q12: How does `asyncio` work under the hood? (Event Loop Architecture)

**Answer:**

1. **The Event Loop:** An infinite loop (`while True`) that monitors file descriptors (sockets) for readability/writability.
2. **Selectors:** Python uses the `selectors` module (which wraps OS-specific APIs like `epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows). The loop tells the OS: "Wake me up when data is ready on any of these 10,000 sockets."
3. **Coroutines as State Machines:** When you `await socket.read()`, the coroutine yields control back to the event loop. The loop registers the socket with `epoll`.
4. **Resumption:** When the OS signals data is ready, the event loop finds the corresponding coroutine and calls `.send(data)` on it, resuming its execution exactly where it paused.

Because of this, `asyncio` can handle tens of thousands of connections on a single thread with minimal RAM (unlike OS threads, which consume ~8MB RAM each).

---

*End of Async & Concurrency — 12 questions covering GIL, Threading vs Processing vs Async, patterns, and internal architecture.*
