## Thread vs Concurrency

These are **not the same thing**.

- **Thread** is an execution unit.
- **Concurrency** is the ability to make progress on multiple tasks at the same time.

---

## Thread

A thread is the smallest unit of execution inside a process.

Example:

```text
Process
   │
   ├── Thread 1
   ├── Thread 2
   └── Thread 3
```

All threads:

- Share the same memory
- Share the same process resources
- Execute independently

Example:

```text
Web Server

Thread 1 → User A request
Thread 2 → User B request
Thread 3 → User C request
```

---

## Concurrency

Concurrency means handling **multiple tasks during the same time period**, even if they are not literally executing simultaneously.

Example:

```text
Task A
↓↓
Task B
↓↓
Task C
```

The CPU switches between tasks quickly.

This gives the appearance that they run together.

---

## Parallelism vs Concurrency

Concurrency:

```text
CPU

Task A
↓↓
Task B
↓↓
Task A
↓↓
Task C
```

One CPU core switches between tasks.

Parallelism:

```text
Core 1 → Task A

Core 2 → Task B

Core 3 → Task C
```

Tasks actually execute at the same time.

---

## Does concurrency require threads?

**No.**

Concurrency can be achieved using:

```text
Threads
Async/Await
Event Loop
Coroutines
Processes
```

Example:

Python AsyncIO:

```python
async def fetch():
    ...
```

No extra threads are created, but multiple requests are handled concurrently.

---

## When to use threads

Use threads for:

```text
HTTP calls
File I/O
Database queries
Network communication
```

---

## When to use multiprocessing

Use processes for:

```text
Image processing
Machine Learning
Video encoding
Heavy computation
```

---

## Interview answer

A thread is an execution unit within a process that shares memory with other threads. Concurrency is a programming model that allows multiple tasks to make progress during the same time period. Threads are one way to achieve concurrency, but concurrency can also be implemented using asynchronous programming or multiple processes.

For I/O-bound applications, I typically use threads or async programming because tasks spend most of their time waiting. For CPU-bound workloads, I use multiprocessing so work can execute in parallel on multiple CPU cores.

> A thread is a mechanism; concurrency is a capability. Threads are one way to implement concurrency, but concurrency can also be achieved without threads using asynchronous programming.