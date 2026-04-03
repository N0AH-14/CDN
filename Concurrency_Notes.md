# ⚙️ Python Concurrency — Comprehensive Notes
> Processes, Threads, GIL, `threading`, `multiprocessing`, `concurrent.futures`, `asyncio` and more.
> Written for Python-proficient readers — no hand-holding, maximum depth.

---

## Table of Contents
1. [Fundamentals — OS Concepts](#1-fundamentals--os-concepts)
2. [The Python GIL — What, Why, and Impact](#2-the-python-gil--what-why-and-impact)
3. [threading — Multithreading](#3-threading--multithreading)
4. [Thread Synchronization Primitives](#4-thread-synchronization-primitives)
5. [multiprocessing — True Parallelism](#5-multiprocessing--true-parallelism)
6. [Inter-Process Communication (IPC)](#6-inter-process-communication-ipc)
7. [Process Synchronization](#7-process-synchronization)
8. [concurrent.futures — High-Level API](#8-concurrentfutures--high-level-api)
9. [asyncio — Cooperative Concurrency](#9-asyncio--cooperative-concurrency)
10. [Choosing the Right Tool](#10-choosing-the-right-tool)
11. [Shared State, Race Conditions & Deadlocks](#11-shared-state-race-conditions--deadlocks)
12. [Thread Pools & Process Pools](#12-thread-pools--process-pools)
13. [subprocess — Spawning External Processes](#13-subprocess--spawning-external-processes)
14. [Advanced Patterns & Real-World Recipes](#14-advanced-patterns--real-world-recipes)
15. [Debugging & Profiling Concurrent Code](#15-debugging--profiling-concurrent-code)
16. [Cheat Sheet Quick Reference](#16-cheat-sheet-quick-reference)

---

## 1. Fundamentals — OS Concepts

Before any Python-specific code, you need a solid mental model of what the OS does.

### Process
- An **independent program in execution** with its own memory space (heap, stack, code, data segments).
- Managed by the OS scheduler.
- Has its own **PID** (process ID).
- Processes are **isolated** — one process cannot read another's memory directly (without IPC).
- **Expensive to create** (fork/exec, copy-on-write pages, OS bookkeeping).
- **Context switching** between processes is slower than between threads.

### Thread
- A **unit of execution within a process**.
- All threads in a process **share the same memory space** (heap, globals, file descriptors).
- Each thread has its own **stack** and **registers**.
- **Cheaper to create** than a process.
- **Context switching** between threads is fast (same address space, no TLB flush).
- Risk: shared memory → race conditions without synchronization.

### Concurrency vs Parallelism
| | Concurrency | Parallelism |
|---|---|---|
| Definition | Multiple tasks make **progress** | Multiple tasks run **simultaneously** |
| Requires | Interleaving / time-slicing | Multiple CPU cores |
| Python example | `asyncio`, `threading` (GIL-limited) | `multiprocessing` |
| Analogy | One cook juggling multiple pots | Multiple cooks each at their own stove |

### CPU-Bound vs I/O-Bound
- **CPU-Bound**: task spends most time computing (matrix math, compression, ML inference).
  - Bottleneck: CPU cycles. Threads don't help (GIL). Use `multiprocessing`.
- **I/O-Bound**: task spends most time waiting (network, disk, DB, API calls).
  - Bottleneck: waiting. Threads or async both help because the GIL is released during I/O.

### The Scheduler
The OS preemptively switches between threads/processes. You have no control over *when* a switch happens — this is the root of race conditions.

---

## 2. The Python GIL — What, Why, and Impact

### What is the GIL?
The **Global Interpreter Lock** is a mutex inside CPython that **allows only one thread to execute Python bytecode at a time**, even on a multi-core machine.

```
Thread 1: ──[runs]──[GIL released for I/O]──────────────────[waits]──[runs]──
Thread 2: ──────────────────────────────────[runs]──[GIL released for I/O]──
```

### Why Does It Exist?
CPython's memory management (specifically **reference counting**) is not thread-safe. Without the GIL, two threads could simultaneously decrement the same object's reference count, causing use-after-free bugs. The GIL was the simplest fix when CPython was first written.

### When Is the GIL Released?
The GIL is NOT held forever. It's released:
- During **I/O operations** (file read/write, socket recv/send, sleep).
- During calls to **C extensions** that explicitly release it (NumPy, Pillow, etc.).
- Every **N bytecode instructions** (controlled by `sys.getswitchinterval()`, default 5ms) for cooperative switching.

```python
import sys
sys.getswitchinterval()   # → 0.005 (seconds)
sys.setswitchinterval(0.001)  # check every 1ms
```

### Impact on Multithreading

| Task Type | Threads Help? | Why |
|---|---|---|
| I/O-bound | ✅ Yes | GIL released during I/O; other threads run |
| CPU-bound | ❌ No | GIL held; threads take turns on ONE core |
| Mixed | ⚠️ Partial | Depends on I/O ratio |

### Bypassing the GIL
1. **`multiprocessing`** — separate processes, separate GILs.
2. **C extensions** — NumPy, SciPy release GIL for their heavy work.
3. **Cython** with `nogil` context.
4. **PyPy** — no GIL (different runtime).
5. **Python 3.13+** — experimental "free-threaded" build (`--disable-gil`).

```python
# Python 3.13+ check
import sys
print(sys._is_gil_enabled())  # False in free-threaded build
```

### Demonstrating the GIL's Effect
```python
import threading, time

COUNT = 50_000_000

def count_down(n):
    while n > 0:
        n -= 1

# Single-threaded
start = time.perf_counter()
count_down(COUNT)
print(f"Single: {time.perf_counter() - start:.2f}s")

# Two threads — NOT faster (GIL serializes them, plus switching overhead)
t1 = threading.Thread(target=count_down, args=(COUNT // 2,))
t2 = threading.Thread(target=count_down, args=(COUNT // 2,))
start = time.perf_counter()
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Two threads: {time.perf_counter() - start:.2f}s")
# Often SLOWER than single-thread due to GIL contention + context switching!
```

---

## 3. threading — Multithreading

```python
import threading
```

### Creating & Starting Threads

#### Method 1: Function-based
```python
def worker(name, delay):
    print(f"Thread {name} starting")
    time.sleep(delay)
    print(f"Thread {name} done")

t = threading.Thread(target=worker, args=('A', 2), kwargs={})
t.start()   # begins execution
t.join()    # block until t finishes
```

#### Method 2: Subclassing Thread
```python
class MyThread(threading.Thread):
    def __init__(self, name, data):
        super().__init__(name=name)
        self.data = data
        self.result = None

    def run(self):           # override run(), NOT start()
        self.result = sum(self.data)

t = MyThread("worker-1", range(100))
t.start()
t.join()
print(t.result)   # → 4950
```

### Thread Properties & Methods
```python
t = threading.Thread(target=worker, args=('A',), daemon=True)

t.start()           # launch thread
t.join()            # wait for completion
t.join(timeout=5)   # wait max 5 seconds

t.is_alive()        # True if still running
t.name              # thread name (auto-assigned or set)
t.ident             # OS thread ID
t.daemon            # True → thread dies when main thread dies
t.daemon = True     # must set BEFORE start()

threading.current_thread()           # the calling thread object
threading.main_thread()              # the main thread
threading.active_count()             # number of alive threads
threading.enumerate()                # list of all alive threads
```

### Daemon Threads
```python
# Daemon threads are abruptly killed when the main program exits.
# Non-daemon threads keep the program alive until they finish.

t = threading.Thread(target=background_task, daemon=True)
t.start()
# Program exits even if t is still running
```

### Thread-Local Storage
Each thread gets its own copy of thread-local variables — safe for things like DB connections.
```python
local_data = threading.local()

def worker():
    local_data.x = threading.current_thread().name  # own copy
    time.sleep(1)
    print(local_data.x)  # safe: reads own copy

threads = [threading.Thread(target=worker) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()
```

### Passing Results Back from Threads
Threads can't return values directly. Solutions:

```python
# Solution 1: Mutable container
results = {}
def worker(key, val):
    results[key] = val * 2

t = threading.Thread(target=worker, args=('a', 10))
t.start(); t.join()
print(results)   # {'a': 20}

# Solution 2: queue.Queue
import queue
q = queue.Queue()
def worker():
    q.put(42)

t = threading.Thread(target=worker)
t.start(); t.join()
print(q.get())  # 42

# Solution 3: Subclass Thread and store result in self (shown above)
```

---

## 4. Thread Synchronization Primitives

When threads share data, you need synchronization to prevent **race conditions**.

### Lock (Mutex)
A **Lock** allows only ONE thread to hold it at a time. All others block on `acquire()`.

```python
lock = threading.Lock()

# Classic usage
lock.acquire()
try:
    shared_counter += 1  # critical section
finally:
    lock.release()

# Preferred: context manager (auto-releases even on exception)
with lock:
    shared_counter += 1
```

#### Demonstrating the Race Condition Without Lock
```python
counter = 0

def increment(n):
    global counter
    for _ in range(n):
        counter += 1   # NOT atomic! read-modify-write

threads = [threading.Thread(target=increment, args=(10000,)) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # LIKELY not 50000 — data race!

# Fix:
lock = threading.Lock()
def safe_increment(n):
    global counter
    for _ in range(n):
        with lock:
            counter += 1
```

### RLock (Re-entrant Lock)
A thread that already holds the lock can `acquire()` it again without deadlocking.
Used when a locked method calls another locked method.
```python
rlock = threading.RLock()

def outer():
    with rlock:
        inner()   # would deadlock with a regular Lock

def inner():
    with rlock:   # same thread can re-acquire RLock
        do_work()
```

### Semaphore
A counter-based lock. Allows **N** threads to enter simultaneously.
```python
sem = threading.Semaphore(3)  # max 3 threads at once

def limited_worker(i):
    with sem:
        print(f"Thread {i} working")
        time.sleep(1)

# Bounded Semaphore: raises ValueError if released more than N times
bsem = threading.BoundedSemaphore(3)
```

**Use case**: Rate-limiting access to a resource (e.g., max 5 concurrent DB connections).

### Event
A simple flag — threads wait until a flag is set.
```python
event = threading.Event()

def waiter():
    print("Waiting for event...")
    event.wait()           # blocks until event is set
    print("Event fired! Continuing.")

def setter():
    time.sleep(2)
    event.set()            # unblocks all waiters

threading.Thread(target=waiter).start()
threading.Thread(target=setter).start()

# Other methods:
event.clear()              # reset flag to False
event.is_set()             # check current state
event.wait(timeout=5)      # wait with timeout, returns bool
```

### Condition
Allows threads to wait for a specific condition while holding a lock.
The classic **producer-consumer** primitive.
```python
condition = threading.Condition()
items = []

def producer():
    for i in range(5):
        with condition:
            items.append(i)
            print(f"Produced {i}")
            condition.notify()      # wake one waiting consumer
        time.sleep(0.5)

def consumer():
    while True:
        with condition:
            while not items:
                condition.wait()    # release lock + sleep until notify()
            item = items.pop(0)
            print(f"Consumed {item}")

threading.Thread(target=producer).start()
threading.Thread(target=consumer, daemon=True).start()

# condition.notify_all() — wake ALL waiting threads
```

### Barrier
All threads must reach the barrier before any can proceed. Used for phased computation.
```python
barrier = threading.Barrier(3)   # wait for 3 threads

def phase_worker(i):
    print(f"Thread {i}: phase 1 done")
    barrier.wait()                 # all 3 must arrive
    print(f"Thread {i}: phase 2 starting")

for i in range(3):
    threading.Thread(target=phase_worker, args=(i,)).start()
```

### Timer
Run a function after a delay (in a new thread).
```python
def remind():
    print("Time's up!")

t = threading.Timer(interval=5.0, function=remind)
t.start()
t.cancel()   # cancel if not yet fired
```

### queue.Queue — Thread-Safe Queue
The `queue` module provides thread-safe FIFO, LIFO, and priority queues.
```python
import queue

q = queue.Queue(maxsize=10)   # 0 = unlimited

q.put(item)              # blocks if full
q.put(item, block=False) # raises queue.Full if full
q.put_nowait(item)       # same as block=False

q.get()                  # blocks if empty
q.get(timeout=2)         # raises queue.Empty after 2s
q.get_nowait()

q.task_done()            # signal that item is processed
q.join()                 # block until all items task_done()

q.qsize()                # approximate size
q.empty()
q.full()

# LIFO Queue (stack)
lq = queue.LifoQueue()

# Priority Queue (min-heap — lower number = higher priority)
pq = queue.PriorityQueue()
pq.put((1, 'high priority'))
pq.put((10, 'low priority'))
pq.get()  # → (1, 'high priority')
```

---

## 5. multiprocessing — True Parallelism

`multiprocessing` spawns separate OS processes, each with its own Python interpreter and GIL — enabling **true CPU parallelism**.

```python
import multiprocessing as mp
```

### Creating Processes

#### Function-based
```python
def worker(name, n):
    print(f"Process {name}, PID: {mp.current_process().pid}")
    return sum(range(n))

if __name__ == '__main__':   # REQUIRED guard on Windows/macOS
    p = mp.Process(target=worker, args=('A', 1000000))
    p.start()
    p.join()
```

**⚠️ Always use `if __name__ == '__main__':` guard.**
Without it, spawning a new process re-imports the module and re-spawns processes recursively on Windows.

#### Subclassing Process
```python
class MyProcess(mp.Process):
    def __init__(self, data):
        super().__init__()
        self.data = data

    def run(self):
        result = sum(self.data)
        print(f"Result: {result}")

if __name__ == '__main__':
    p = MyProcess(range(100))
    p.start()
    p.join()
```

### Process Properties & Methods
```python
p.start()
p.join()
p.join(timeout=5)
p.terminate()      # sends SIGTERM — process may not clean up
p.kill()           # sends SIGKILL — immediate
p.is_alive()
p.pid              # OS process ID
p.exitcode         # None if running; 0 = success; negative = signal
p.name
p.daemon           # daemon process: killed when parent exits
```

### Start Methods
How a new process is spawned differs by OS:

```python
# spawn  — default on Windows/macOS. Fresh Python interpreter. Safest.
# fork   — default on Linux. Copies parent memory via COW. Fast but unsafe with threads.
# forkserver — hybrid. Uses a server process to fork from.

mp.set_start_method('spawn')   # call once, before any Process creation

# Or per-context:
ctx = mp.get_context('spawn')
p = ctx.Process(target=worker)
```

---

## 6. Inter-Process Communication (IPC)

Processes have **separate memory**, so they need explicit channels to communicate.

### Queue (multiprocessing.Queue)
```python
from multiprocessing import Process, Queue

def producer(q):
    for i in range(5):
        q.put(i)
    q.put(None)   # sentinel to signal done

def consumer(q):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Got: {item}")

if __name__ == '__main__':
    q = Queue()
    p1 = Process(target=producer, args=(q,))
    p2 = Process(target=consumer, args=(q,))
    p1.start(); p2.start()
    p1.join(); p2.join()
```

### Pipe
A two-way communication channel. Returns a pair of `Connection` objects.
```python
from multiprocessing import Process, Pipe

def child(conn):
    conn.send("Hello from child")
    msg = conn.recv()
    print(f"Child received: {msg}")
    conn.close()

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=child, args=(child_conn,))
    p.start()
    print(parent_conn.recv())         # blocks until child sends
    parent_conn.send("Hello from parent")
    p.join()

# Pipe(duplex=False) → one-directional (parent reads, child writes)
```

### Shared Memory — `Value` and `Array`
Shared C-type objects in shared memory. Require explicit locks for safety.
```python
from multiprocessing import Process, Value, Array
import ctypes

def worker(shared_val, shared_arr):
    with shared_val.get_lock():
        shared_val.value += 1
    for i in range(len(shared_arr)):
        shared_arr[i] *= 2

if __name__ == '__main__':
    val = Value(ctypes.c_int, 0)          # shared integer
    arr = Array(ctypes.c_double, [1.0, 2.0, 3.0])  # shared array

    procs = [Process(target=worker, args=(val, arr)) for _ in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()

    print(val.value)   # 4
    print(list(arr))   # [16.0, 32.0, 48.0]  (doubled 4 times)
```

### `multiprocessing.shared_memory` (Python 3.8+)
Raw shared memory blocks — fastest IPC, works with NumPy arrays.
```python
from multiprocessing import shared_memory
import numpy as np

if __name__ == '__main__':
    # Create
    shm = shared_memory.SharedMemory(create=True, size=400)
    arr = np.ndarray((100,), dtype=np.float32, buffer=shm.buf)
    arr[:] = np.arange(100)

    # In another process: attach to it by name
    existing_shm = shared_memory.SharedMemory(name=shm.name)
    arr2 = np.ndarray((100,), dtype=np.float32, buffer=existing_shm.buf)
    print(arr2[0])  # 0.0

    existing_shm.close()
    shm.close()
    shm.unlink()   # delete the underlying shared memory
```

### Manager (Proxy Objects)
Managers create server processes that host shared objects. Slower but more flexible — support dicts, lists, etc.
```python
from multiprocessing import Manager, Process

def worker(shared_dict, key, val):
    shared_dict[key] = val

if __name__ == '__main__':
    with Manager() as manager:
        d = manager.dict()
        procs = [Process(target=worker, args=(d, i, i**2)) for i in range(5)]
        for p in procs: p.start()
        for p in procs: p.join()
        print(dict(d))   # {0:0, 1:1, 2:4, 3:9, 4:16}

    # manager.list(), manager.Queue(), manager.Value(), manager.Lock() also available
```

---

## 7. Process Synchronization

```python
from multiprocessing import Lock, RLock, Semaphore, Event, Barrier, Condition

# Same API as threading primitives — but work across processes
lock = mp.Lock()
with lock:
    # critical section
    ...

event = mp.Event()
event.set()
event.wait()
event.clear()

sem = mp.Semaphore(3)
barrier = mp.Barrier(4)
```

**Key difference**: `multiprocessing` primitives are backed by OS-level synchronization (semaphores, mutexes) since processes don't share memory.

---

## 8. concurrent.futures — High-Level API

`concurrent.futures` provides a clean, unified interface over both threads and processes.

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed, wait
```

### ThreadPoolExecutor
```python
def fetch(url):
    import urllib.request
    with urllib.request.urlopen(url) as r:
        return len(r.read())

urls = ['https://python.org', 'https://pypi.org', 'https://docs.python.org']

# submit() — returns a Future immediately
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(fetch, url) for url in urls]
    for future in futures:
        print(future.result())   # blocks until done

# map() — like built-in map, but concurrent; maintains order
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch, urls))  # blocks until all done
    results = list(executor.map(fetch, urls, timeout=10))
```

### ProcessPoolExecutor
```python
def cpu_task(n):
    return sum(i * i for i in range(n))

numbers = [10**6, 2*10**6, 3*10**6]

if __name__ == '__main__':
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(cpu_task, numbers))
    print(results)
```

### Future Object
```python
future = executor.submit(some_func, arg1, arg2)

future.result()            # block and return result (raises exception if task raised)
future.result(timeout=5)   # raises TimeoutError if not done in 5s
future.exception()         # returns the exception (or None)
future.done()              # True if finished
future.running()           # True if currently executing
future.cancelled()         # True if cancelled
future.cancel()            # attempt to cancel (only works if not yet started)

future.add_done_callback(fn)  # fn(future) called when done
```

### `as_completed()` — Process Results as They Finish
```python
with ProcessPoolExecutor() as executor:
    futures = {executor.submit(cpu_task, n): n for n in numbers}

    for future in as_completed(futures):
        n = futures[future]    # look up original input
        try:
            result = future.result()
            print(f"n={n} → {result}")
        except Exception as e:
            print(f"n={n} failed: {e}")
```

### `wait()` — Wait with Fine-Grained Control
```python
from concurrent.futures import wait, FIRST_COMPLETED, ALL_COMPLETED, FIRST_EXCEPTION

done, not_done = wait(futures, return_when=FIRST_COMPLETED)
done, not_done = wait(futures, timeout=5)
done, not_done = wait(futures, return_when=ALL_COMPLETED)
```

### Initializer — Setup Worker State
```python
def init_worker(shared_resource):
    global resource
    resource = shared_resource   # each worker gets this on startup

with ProcessPoolExecutor(max_workers=4,
                         initializer=init_worker,
                         initargs=(some_resource,)) as executor:
    ...
```

---

## 9. asyncio — Cooperative Concurrency

`asyncio` is **single-threaded** concurrency using coroutines and an event loop.
It does NOT use threads or processes — it uses **cooperative scheduling** (tasks voluntarily yield at `await` points).

```python
import asyncio
```

### Core Concepts
- **Coroutine**: a function defined with `async def`. Calling it returns a coroutine object, not a result.
- **Task**: a coroutine wrapped in a `Task` object and scheduled on the event loop.
- **Event Loop**: the engine that runs tasks, switching between them at `await` points.
- **`await`**: suspends the current coroutine, lets the event loop run others.

### Basic Coroutine
```python
async def greet(name, delay):
    await asyncio.sleep(delay)   # yields control; non-blocking
    print(f"Hello, {name}!")
    return f"done-{name}"

# Run coroutines
asyncio.run(greet("Alice", 1))   # Python 3.7+

# Run multiple concurrently
async def main():
    result = await greet("Alice", 1)   # awaits one at a time — sequential!
    print(result)

asyncio.run(main())
```

### Running Concurrently with `gather`
```python
async def main():
    # All three start simultaneously; total time ≈ max(delays), not sum
    results = await asyncio.gather(
        greet("Alice", 1),
        greet("Bob",   2),
        greet("Carol", 0.5)
    )
    print(results)   # ['done-Alice', 'done-Bob', 'done-Carol']

asyncio.run(main())
```

### `asyncio.create_task()` — Schedule Without Awaiting Immediately
```python
async def main():
    task1 = asyncio.create_task(greet("Alice", 1))
    task2 = asyncio.create_task(greet("Bob", 2))
    # Both tasks start NOW (scheduled)
    await asyncio.sleep(0)     # give event loop a chance to start them
    result1 = await task1
    result2 = await task2
```

### `asyncio.TaskGroup` (Python 3.11+) — Preferred Modern Syntax
```python
async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(greet("Alice", 1))
        t2 = tg.create_task(greet("Bob", 2))
    # All tasks finished by here
    print(t1.result(), t2.result())
```

### Timeouts
```python
async def slow():
    await asyncio.sleep(10)

async def main():
    try:
        await asyncio.wait_for(slow(), timeout=2.0)
    except asyncio.TimeoutError:
        print("Timed out!")
```

### asyncio vs threading for I/O

```python
# asyncio approach — thousands of concurrent connections in ONE thread
async def fetch_all(urls):
    import aiohttp
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

async def fetch(session, url):
    async with session.get(url) as resp:
        return await resp.text()
```

### Running CPU Work from asyncio (without blocking the loop)
```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n):
    return sum(i*i for i in range(n))

async def main():
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10**7)
    print(result)

# For thread executor (I/O wrappers):
async def main():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_io_func, arg)
```

### Async Queues
```python
async def producer(q):
    for i in range(5):
        await q.put(i)
        await asyncio.sleep(0.1)
    await q.put(None)  # sentinel

async def consumer(q):
    while True:
        item = await q.get()
        if item is None:
            break
        print(f"Got: {item}")
        q.task_done()

async def main():
    q = asyncio.Queue()
    await asyncio.gather(producer(q), consumer(q))

asyncio.run(main())
```

### Async Context Managers and Iterators
```python
class AsyncResource:
    async def __aenter__(self):
        await asyncio.sleep(0)   # simulate async setup
        return self

    async def __aexit__(self, *args):
        await asyncio.sleep(0)   # simulate async teardown

async def use():
    async with AsyncResource() as r:
        print("Using resource")

# Async iterator
class AsyncCounter:
    def __init__(self, stop):
        self.stop = stop
        self.n = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.n >= self.stop:
            raise StopAsyncIteration
        self.n += 1
        await asyncio.sleep(0.1)
        return self.n

async def main():
    async for val in AsyncCounter(5):
        print(val)
```

---

## 10. Choosing the Right Tool

```
Is your task I/O-bound or CPU-bound?
│
├── I/O-Bound (network, disk, DB, APIs)
│   ├── Few concurrent tasks, simple code  →  threading
│   ├── Many concurrent tasks (100s–1000s) →  asyncio  ← preferred
│   └── Mixing sync & async               →  threading + run_in_executor
│
└── CPU-Bound (computation, parsing, ML)
    ├── Pure Python work                  →  multiprocessing
    ├── NumPy / C-extension work          →  threading (GIL released by C code)
    └── Mix of both                       →  ProcessPoolExecutor
```

| Scenario | Best Tool |
|---|---|
| Web scraping 100 URLs | `asyncio` + `aiohttp` |
| Download 10 files | `ThreadPoolExecutor` |
| Resize 1000 images (Pillow) | `ThreadPoolExecutor` (Pillow releases GIL) |
| Compress 1000 files (pure Python) | `ProcessPoolExecutor` |
| Real-time chat server | `asyncio` |
| Data pipeline stages | `multiprocessing.Queue` |
| Parallel numerical computation | `multiprocessing` or `numpy` (C-level) |
| Background task in web app | `threading` (simple) or `asyncio` (async frameworks) |
| Subprocess shell commands | `subprocess` |

---

## 11. Shared State, Race Conditions & Deadlocks

### Race Condition
Occurs when the result depends on the **interleaving order** of threads.

```python
# The "check-then-act" race:
if key not in cache:        # Thread A checks: not in cache
    # ← Thread B also checks here: also not in cache
    cache[key] = compute()  # Both compute and overwrite!

# Fix: atomic check-and-set with a Lock
with cache_lock:
    if key not in cache:
        cache[key] = compute()
```

### Deadlock
Two or more threads waiting for each other's lock — indefinitely.

```python
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:      # waits for lock_b
            print("T1 done")

def thread2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:      # waits for lock_a — DEADLOCK
            print("T2 done")
```

**Prevention strategies:**
1. **Lock ordering** — always acquire locks in the same global order.
2. **Try-lock with timeout** — use `lock.acquire(timeout=1)`.
3. **Single coarse-grained lock** — one lock for everything (reduces parallelism but safe).
4. **Avoid nested locks** — restructure code to not need multiple locks.

```python
# Fix using consistent ordering
def thread1():
    with lock_a:
        with lock_b:    # always acquire a then b
            ...

def thread2():
    with lock_a:        # same order: a then b
        with lock_b:
            ...
```

### Livelock
Threads keep responding to each other but make no progress. Rare but possible.

### Starvation
A thread never gets CPU time because other threads always "win" the scheduler.

### Atomic Operations in Python
Some operations are **effectively atomic** due to the GIL, but **don't rely on this**:
```python
# These are safe in CPython (single bytecode op):
x = y         # reference assignment
x += 1        # NOT safe for shared globals (read-modify-write is 3 ops)
list.append() # safe
dict[key] = val  # safe for simple assignment
```
When in doubt, use a Lock.

---

## 12. Thread Pools & Process Pools

### `multiprocessing.Pool` (Lower-Level than concurrent.futures)
```python
from multiprocessing import Pool

def square(x):
    return x * x

if __name__ == '__main__':
    with Pool(processes=4) as pool:
        # map — blocks until all done, returns in order
        results = pool.map(square, range(10))

        # imap — lazy iterator, results in input order
        for result in pool.imap(square, range(10)):
            print(result)

        # imap_unordered — results as they complete (faster if order doesn't matter)
        for result in pool.imap_unordered(square, range(10)):
            print(result)

        # starmap — multiple arguments
        results = pool.starmap(pow, [(2, 3), (3, 2), (4, 5)])

        # apply_async — non-blocking single call
        r = pool.apply_async(square, (5,))
        print(r.get())   # blocks here

        # map_async — non-blocking map
        async_result = pool.map_async(square, range(10))
        results = async_result.get(timeout=10)
```

### Chunksize Optimization
For large iterables, `chunksize` groups items to reduce IPC overhead:
```python
pool.map(func, range(10**6), chunksize=1000)
# Sends 1000 items per IPC message instead of 1
```

---

## 13. subprocess — Spawning External Processes

`subprocess` runs shell commands or external programs from Python.

```python
import subprocess
```

### `subprocess.run()` — Preferred High-Level API
```python
result = subprocess.run(['ls', '-la'])          # inherits stdout/stderr
result = subprocess.run(['echo', 'hello'], capture_output=True, text=True)

result.returncode   # 0 = success
result.stdout       # captured stdout (string if text=True)
result.stderr       # captured stderr
result.args         # command that was run

# Raise on non-zero exit
result = subprocess.run(['false'], check=True)  # raises CalledProcessError
```

### Common Options
```python
subprocess.run(
    ['cmd', 'arg'],
    capture_output=True,     # capture stdout and stderr
    text=True,               # decode as string (vs bytes)
    input="stdin data",      # send to stdin
    timeout=30,              # raise TimeoutExpired
    check=True,              # raise CalledProcessError on failure
    cwd='/tmp',              # working directory
    env={'PATH': '/usr/bin'},# custom env vars
    shell=True               # run via shell (SECURITY RISK with user input!)
)
```

### `subprocess.Popen` — Full Control
```python
# Start process without waiting
proc = subprocess.Popen(
    ['python', '-c', 'import time; time.sleep(5)'],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

proc.pid            # PID
proc.poll()         # None if running, else returncode
stdout, stderr = proc.communicate(timeout=10)  # wait + get output
proc.wait()         # just wait for exit
proc.terminate()    # SIGTERM
proc.kill()         # SIGKILL
```

### Piping Commands
```python
# Equivalent to: ls -la | grep .py
p1 = subprocess.Popen(['ls', '-la'], stdout=subprocess.PIPE)
p2 = subprocess.Popen(['grep', '.py'], stdin=p1.stdout, stdout=subprocess.PIPE)
p1.stdout.close()   # allow p1 to receive SIGPIPE if p2 exits
output = p2.communicate()[0]
```

---

## 14. Advanced Patterns & Real-World Recipes

### Producer-Consumer with a Thread Pool
```python
import queue, threading, time

def producer(q, items):
    for item in items:
        q.put(item)
    for _ in range(NUM_WORKERS):
        q.put(None)  # one sentinel per worker

def consumer(q, results):
    while True:
        item = q.get()
        if item is None:
            break
        results.append(item * 2)
        q.task_done()

NUM_WORKERS = 4
q = queue.Queue(maxsize=20)
results = []

prod = threading.Thread(target=producer, args=(q, range(100)))
workers = [threading.Thread(target=consumer, args=(q, results)) for _ in range(NUM_WORKERS)]

prod.start()
for w in workers: w.start()
prod.join()
for w in workers: w.join()
print(sorted(results))
```

### Map-Reduce with multiprocessing
```python
from multiprocessing import Pool

def map_func(chunk):
    return sum(x * x for x in chunk)

def chunks(lst, n):
    for i in range(0, len(lst), n):
        yield lst[i:i+n]

if __name__ == '__main__':
    data = list(range(10**6))
    chunk_size = 10**5
    data_chunks = list(chunks(data, chunk_size))

    with Pool() as pool:
        partial_sums = pool.map(map_func, data_chunks)

    total = sum(partial_sums)   # reduce step
    print(total)
```

### Fan-Out / Fan-In with Futures
```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def analyze(data_chunk):
    # CPU-bound analysis
    return {'mean': sum(data_chunk)/len(data_chunk), 'n': len(data_chunk)}

if __name__ == '__main__':
    chunks = [range(i, i+1000) for i in range(0, 10000, 1000)]

    results = []
    with ProcessPoolExecutor(max_workers=4) as executor:
        future_to_chunk = {executor.submit(analyze, list(c)): c for c in chunks}
        for future in as_completed(future_to_chunk):
            results.append(future.result())

    # Combine
    total_n = sum(r['n'] for r in results)
    overall_mean = sum(r['mean'] * r['n'] for r in results) / total_n
```

### Timeout Pattern for Threads
(Threads can't be killed from outside — use an Event flag)
```python
def cancellable_worker(stop_event, data):
    for item in data:
        if stop_event.is_set():
            print("Cancelled!")
            return
        process(item)

stop = threading.Event()
t = threading.Thread(target=cancellable_worker, args=(stop, big_list))
t.start()
time.sleep(5)
stop.set()    # politely ask thread to stop
t.join()
```

### Async HTTP Requests (aiohttp)
```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
        return await resp.json()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return results

urls = [f'https://jsonplaceholder.typicode.com/posts/{i}' for i in range(1, 20)]
results = asyncio.run(main(urls))
```

### Thread-Safe Singleton
```python
class Singleton:
    _instance = None
    _lock = threading.Lock()

    @classmethod
    def get_instance(cls):
        if cls._instance is None:         # first check (no lock, fast)
            with cls._lock:
                if cls._instance is None:  # second check (under lock)
                    cls._instance = cls()
        return cls._instance
```
This is the **double-checked locking** pattern.

### Rate Limiter using Semaphore
```python
class RateLimiter:
    def __init__(self, max_calls, period):
        self.semaphore = threading.Semaphore(max_calls)
        self.period = period

    def __enter__(self):
        self.semaphore.acquire()
        return self

    def __exit__(self, *args):
        threading.Timer(self.period, self.semaphore.release).start()

limiter = RateLimiter(max_calls=5, period=1.0)  # max 5 calls per second

def api_call(url):
    with limiter:
        return requests.get(url)
```

---

## 15. Debugging & Profiling Concurrent Code

### Logging in Threads/Processes (Safe)
```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s %(threadName)s %(processName)s %(message)s'
)
log = logging.getLogger(__name__)

def worker():
    log.debug("Starting")
    log.info("Done")
```
`logging` module is **thread-safe** in Python.

### Timing Concurrent Code
```python
import time

start = time.perf_counter()
# ... work ...
elapsed = time.perf_counter() - start
print(f"Elapsed: {elapsed:.4f}s")
```

### Detecting Deadlocks
```python
# faulthandler dumps all thread stacks on SIGABRT or after timeout
import faulthandler
faulthandler.enable()
faulthandler.dump_traceback_later(timeout=10, repeat=True)
```

### `threading.settrace()` — Tracing Thread Execution
```python
import sys

def trace(frame, event, arg):
    print(f"{threading.current_thread().name}: {event} in {frame.f_code.co_filename}")
    return trace

threading.settrace(trace)
```

### Profiling Multiprocessed Code
```python
import cProfile, pstats, io

def profiled_worker():
    pr = cProfile.Profile()
    pr.enable()
    actual_work()
    pr.disable()
    s = io.StringIO()
    pstats.Stats(pr, stream=s).sort_stats('cumulative').print_stats(20)
    print(s.getvalue())

# Use in each process, since cProfile is per-process
```

### Common Pitfalls & Fixes

| Pitfall | Symptom | Fix |
|---|---|---|
| No `if __name__ == '__main__'` | RecursionError on Windows | Add the guard |
| Mutating shared list without lock | Inconsistent results | Use `Lock` or `Queue` |
| Large object in multiprocessing queue | Slow / MemoryError | Use `shared_memory` or `mmap` |
| `apply()` in multiprocessing | `PicklingError` | Functions must be picklable (module-level) |
| Lambda in `ProcessPoolExecutor` | `AttributeError: Can't pickle <lambda>` | Use `functools.partial` or named function |
| Forgetting to `.join()` | Process/thread leaks | Always join or use context managers |
| Blocking the asyncio event loop | Everything freezes | Use `run_in_executor` for sync/blocking work |
| Catching all exceptions silently | Bugs hidden | Re-raise or log exceptions in workers |

### Lambdas and Pickle (multiprocessing gotcha)
```python
# WRONG — lambdas can't be pickled
with ProcessPoolExecutor() as ex:
    ex.map(lambda x: x*2, range(10))  # PicklingError

# CORRECT — use a named function or functools.partial
from functools import partial

def multiply(x, factor):
    return x * factor

double = partial(multiply, factor=2)
with ProcessPoolExecutor() as ex:
    results = list(ex.map(double, range(10)))
```

---

## 16. Cheat Sheet Quick Reference

### Threading
| Task | Code |
|---|---|
| Create thread | `t = threading.Thread(target=fn, args=(a,))` |
| Start thread | `t.start()` |
| Wait for thread | `t.join()` |
| Daemon thread | `t.daemon = True` (before start) |
| Current thread | `threading.current_thread()` |
| Thread count | `threading.active_count()` |
| Mutual exclusion | `with threading.Lock(): ...` |
| Re-entrant lock | `threading.RLock()` |
| Limit concurrency | `threading.Semaphore(N)` |
| Signal an event | `event.set()` / `event.wait()` |
| Producer-consumer | `queue.Queue()` |
| Thread-local data | `threading.local()` |

### Multiprocessing
| Task | Code |
|---|---|
| Create process | `p = mp.Process(target=fn, args=(a,))` |
| Start/wait | `p.start()` / `p.join()` |
| PID | `p.pid` |
| Kill process | `p.terminate()` / `p.kill()` |
| Shared queue | `mp.Queue()` |
| Shared pipe | `parent, child = mp.Pipe()` |
| Shared value | `mp.Value('i', 0)` |
| Shared array | `mp.Array('d', [1.0, 2.0])` |
| Manager dict | `mp.Manager().dict()` |
| Process pool | `mp.Pool(N)` |

### concurrent.futures
| Task | Code |
|---|---|
| Thread pool | `ThreadPoolExecutor(max_workers=N)` |
| Process pool | `ProcessPoolExecutor(max_workers=N)` |
| Submit task | `future = executor.submit(fn, arg)` |
| Map over iterable | `executor.map(fn, iterable)` |
| Get result | `future.result()` |
| Results as ready | `as_completed(futures)` |
| Wait for subset | `wait(futures, return_when=...)` |

### asyncio
| Task | Code |
|---|---|
| Define coroutine | `async def fn(): ...` |
| Await coroutine | `await fn()` |
| Run event loop | `asyncio.run(main())` |
| Run concurrently | `await asyncio.gather(c1, c2)` |
| Create task | `asyncio.create_task(coro)` |
| Async sleep | `await asyncio.sleep(1)` |
| Timeout | `await asyncio.wait_for(coro, timeout=5)` |
| Async queue | `asyncio.Queue()` |
| Run sync in thread | `await loop.run_in_executor(None, fn)` |
| Run CPU in process | `await loop.run_in_executor(pool, fn)` |

---

### Mental Model Summary
```
GIL        → Only one thread runs Python bytecode at a time (CPython)
threading  → Concurrency, NOT parallelism (good for I/O-bound work)
multiprocessing → True parallelism (good for CPU-bound work, separate GIL per process)
asyncio    → Single-threaded concurrency via cooperative scheduling (best for high-concurrency I/O)
concurrent.futures → Clean unified API over threads and processes
subprocess → Spawn and control external OS processes
```

---

*Master these tools and you'll handle everything from simple background threads to massively parallel data pipelines and high-throughput async servers.*
