## GIL (Global Interpreter Lock)

The **Global Interpreter Lock (GIL)** is a mechanism in **CPython** (the standard Python interpreter) that allows **only one thread to execute Python bytecode at a time** within a single process.

---

### Why does Python have a GIL?

The GIL simplifies memory management and makes Python's reference counting thread-safe.

Instead of multiple threads modifying Python objects simultaneously, the GIL ensures only one thread executes Python code at a time.

---

### How it works

Suppose you have:

```python
import threading

def task():
    for _ in range(1000000):
        pass

t1 = threading.Thread(target=task)
t2 = threading.Thread(target=task)

t1.start()
t2.start()
```

Although there are two threads:

```text
Thread 1
Thread 2
```

Only **one thread executes Python bytecode at any instant** because of the GIL.

---

### CPU-bound workload

Example:

```text
Image processing
Mathematical computation
Encryption
Data processing
```

Multiple Python threads **do not run in parallel** on multiple CPU cores.

Solution:

```text
multiprocessing
ProcessPoolExecutor
```

Each process has its own Python interpreter and its own GIL.

---

### I/O-bound workload

Example:

```text
HTTP requests
Database queries
File I/O
Network calls
```

While one thread waits for I/O, it releases the GIL, allowing another thread to run.

Therefore, **multithreading works well for I/O-bound applications**.

---

### CPU vs I/O

| Workload | Best Option |
|----------|-------------|
| CPU-bound | Multiprocessing |
| I/O-bound | Multithreading / Asyncio |

---

### Example

#### CPU-bound

```python
from multiprocessing import Pool

def square(x):
    return x * x

with Pool(4) as p:
    print(p.map(square, range(10)))
```

Uses multiple CPU cores.

---

#### I/O-bound

```python
import threading
import requests

def fetch():
    requests.get("https://example.com")

threads = [threading.Thread(target=fetch) for _ in range(10)]

for t in threads:
    t.start()

for t in threads:
    t.join()
```

Here, threads are beneficial because most of the time is spent waiting for network responses.

---

## Interview answer

The Global Interpreter Lock (GIL) is a lock in CPython that allows only one thread to execute Python bytecode at a time within a process. It exists to simplify memory management and make reference counting thread-safe.

Because of the GIL, multithreading does not improve performance for CPU-bound tasks such as numerical computation. For those workloads, I would use multiprocessing so each process has its own interpreter and GIL. However, for I/O-bound workloads like API calls, database operations, and file access, multithreading works well because threads release the GIL while waiting for I/O.


> The GIL limits parallel execution of Python bytecode in a single process, so we use **threads for I/O-bound workloads** and **multiple processes for CPU-bound workloads**.