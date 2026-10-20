### Explain all these - concurrency, Linux process/memory, cgroups v1/v2, namespaces, non-blocking I/O.

These topics fit together nicely at the **OS/runtime level**. A useful mental model is:

> **Concurrency** = many things making progress  
> **Processes/memory** = how Linux runs and isolates programs  
> **Namespaces** = what a process can *see*  
> **cgroups** = how much a process can *use*  
> **Non-blocking I/O** = how one thread efficiently handles many I/O operations

These are also the foundations behind containers, Kubernetes, Nginx, Node.js, databases, and high-performance servers.

## 1. Concurrency

**Concurrency means multiple tasks are in progress during overlapping periods of time.**

Suppose a server receives:

```text
Request A ────────────────>
Request B    ────────────────>
Request C       ─────────────>
```

They don't necessarily execute CPU instructions simultaneously.

### Concurrency vs parallelism

**Concurrency:**

```text
CPU:
A A A | B B | A | C C | B ...
```

One CPU core switches between tasks.

**Parallelism:**

```text
Core 1: A A A A A
Core 2: B B B B B
Core 3: C C C C C
```

Tasks literally execute at the same time.

So:

```text
Concurrency → structure many tasks so they can make progress
Parallelism → execute multiple tasks simultaneously
```

### Processes vs threads

A Linux program can use multiple **processes**:

```text
nginx master
   ├── worker process
   ├── worker process
   └── worker process
```

Each process has its own virtual address space.

Or multiple **threads**:

```text
Java process
   ├── thread 1
   ├── thread 2
   ├── thread 3
   └── thread 4
```

Threads share the process's memory.

That makes threads cheaper for communication but introduces synchronization problems:

```text
Thread A ──┐
           ├── shared variable
Thread B ──┘
```

Potential problems include race conditions, deadlocks, lock contention, starvation, and visibility/order issues.

---

# 2. Linux processes

A **process** is basically a running instance of a program plus the execution state Linux maintains for it.

For example:

```bash
./myserver
```

Linux creates a process with things like:

```text
PID
virtual memory
register state
open file descriptors
environment variables
credentials
threads
signal handlers
resource accounting
```

You can inspect processes with:

```bash
ps aux
top
htop
```

Conceptually:

```text
Linux kernel
│
├── Process 100
│     ├── virtual memory
│     ├── file descriptors
│     └── threads
│
├── Process 101
│     ├── virtual memory
│     └── threads
│
└── Process 102
```

The Linux scheduler decides which runnable thread gets CPU time.

---

# 3. Linux process memory

One of the most important ideas:

**A process does not normally work directly with physical RAM addresses.**

It sees **virtual memory**.

Imagine:

```text
Process virtual address space

high address
┌──────────────────┐
│ Stack            │
│ ↓                │
│                  │
│                  │
│ ↑                │
│ Heap             │
├──────────────────┤
│ Data / BSS       │
├──────────────────┤
│ Program code     │
└──────────────────┘
low address
```

The CPU's MMU and kernel page tables map virtual pages to physical memory.

For example:

```text
Process virtual memory        Physical RAM

0x1000 ────────────────→      RAM page 53
0x2000 ────────────────→      RAM page 918
0x3000 ────────────────→      RAM page 201
```

This gives processes isolation.

Process A can have:

```text
0x1000
```

and Process B can also have:

```text
0x1000
```

while those addresses map to completely different physical RAM.

### Important memory concepts

**Stack** stores things such as function call frames and local variables.

**Heap** is dynamically allocated memory, such as from `malloc()`.

**RSS (Resident Set Size)** roughly represents pages of a process currently resident in RAM.

**Virtual memory size** is the address space mapped/reserved by the process and does **not** mean all of it occupies RAM.

**Page fault** occurs when the CPU accesses a virtual page and the required mapping/page isn't immediately available.

Not every page fault is catastrophic. Minor page faults are extremely common.

---

# 4. Namespaces

Now we're getting into containers.

Linux namespaces answer:

> **What resources can this process see?**

Without namespaces:

```text
Process
   ↓
host processes
host network
host filesystem/mounts
host hostname
...
```

With namespaces:

```text
Process
   ↓
isolated view of resources
```

Important Linux namespaces include:

| Namespace | Isolates |
|---|---|
| PID | process IDs |
| NET | network stack |
| MNT | mount points |
| UTS | hostname/domain |
| IPC | IPC resources |
| USER | users/UIDs |
| CGROUP | cgroup view |
| TIME | clocks |

### PID namespace

Inside a container:

```bash
ps
```

might show:

```text
PID 1  myapp
PID 7  worker
```

But on the host those processes could actually be:

```text
PID 18372 myapp
PID 18401 worker
```

Same underlying processes, different namespace view.

### Network namespace

A network namespace can have its own:

```text
interfaces
IP addresses
routing table
firewall state
ports
```

This is fundamental to container networking.

---

# 5. cgroups

Namespaces answer:

> What can you see?

**cgroups answer:**

> How much can you use?

cgroup = **control group**.

Linux can group processes and control/account for resources such as:

```text
CPU
memory
I/O
process count
```

Conceptually:

```text
Linux host
│
├── cgroup A
│    CPU limit: 2 cores
│    Memory: 4 GB
│
└── cgroup B
     CPU limit: 500m
     Memory: 1 GB
```

This is a huge part of how container resource limits work.

For example, Kubernetes:

```yaml
resources:
  limits:
    cpu: "2"
    memory: "1Gi"
```

eventually relies heavily on Linux cgroups underneath.

---

# 6. cgroups v1

cgroups v1 evolved around separate **controllers**.

You could effectively have different hierarchies for different resource controllers.

Conceptually:

```text
CPU hierarchy
/
├── app1
└── app2

Memory hierarchy
/
├── teamA
│   └── app1
└── app2
```

Controllers included things such as:

```text
cpu
cpuacct
memory
blkio
pids
cpuset
```

This flexibility also made things complicated.

Different controllers could organize processes differently.

---

# 7. cgroups v2

cgroups v2 redesigned the model around a **unified hierarchy**.

Instead of:

```text
CPU hierarchy
Memory hierarchy
IO hierarchy
...
```

you get essentially:

```text
/
├── system.slice
├── user.slice
└── workloads
      ├── app1
      └── app2
```

and controllers operate through that unified tree.

Some common v2 files:

```text
cpu.max
cpu.weight

memory.max
memory.current

io.max

pids.max
```

For example:

```text
memory.max = 1073741824
```

means a cgroup's memory limit is approximately **1 GiB**.

### v1 vs v2 interview answer

Remember this:

> **cgroups v1 allows separate controller hierarchies, while cgroups v2 provides one unified hierarchy with cleaner and more consistent resource management semantics.**

Modern Linux/container environments increasingly use v2.

---

# 8. Namespaces + cgroups = containers

This is probably the most important connection.

A container isn't fundamentally a tiny VM.

At the Linux level, it's largely:

```text
Process
   │
   ├── namespaces
   │      ↓
   │   isolated view
   │
   └── cgroups
          ↓
       resource controls
```

Add filesystem/container-image machinery, capabilities, seccomp, networking, runtime management, etc., and you get the container experience.

Compare with a VM:

```text
VM:
hardware
  ↓
hypervisor
  ↓
guest kernel
  ↓
process

Container:
host kernel
  ↓
isolated process
```

Containers generally **share the host Linux kernel**.

---

# 9. Blocking I/O

Suppose your program performs:

```c
read(socket, buffer, 4096);
```

but no network data has arrived.

With traditional blocking I/O:

```text
Thread
  |
  | read()
  ↓
BLOCKED
  |
  | waiting for network
  |
  ↓
data arrives
  |
RUNNABLE
```

The thread can't continue that operation until data becomes available.

For a server with many connections, a simplistic architecture might require:

```text
Connection 1 → Thread 1
Connection 2 → Thread 2
Connection 3 → Thread 3
...
Connection 10000 → Thread 10000
```

That can become expensive because threads consume memory and scheduling resources.

---

# 10. Non-blocking I/O

With non-blocking I/O, you tell the kernel:

> If the operation cannot complete now, don't put my thread to sleep waiting for it.

A read might effectively respond:

```text
No data available right now.
Try again later.
```

On Linux, you'll commonly encounter:

```text
EAGAIN
EWOULDBLOCK
```

But repeatedly doing:

```text
read()
read()
read()
read()
```

would waste CPU.

So we need an efficient way for the kernel to tell us:

> These sockets are ready.

That's where mechanisms such as `epoll` come in.

---

# 11. epoll

Linux `epoll` lets a program monitor many file descriptors efficiently.

Conceptually:

```text
10,000 connections
        │
        ↓
      epoll
        │
        ↓
"socket 52 and socket 819 are ready"
        │
        ↓
application handles them
```

The application can therefore use relatively few threads to manage huge numbers of connections.

Simplified event loop:

```text
while true:

    events = epoll_wait()

    for event in events:

        if socket readable:
            read()

        if socket writable:
            write()
```

This architecture powers many high-concurrency systems.

Examples include event-driven servers and runtimes such as Nginx and Node.js/libuv.

---

# 12. Non-blocking does NOT mean asynchronous

This distinction is excellent interview material.

**Blocking:**

```text
read()
  ↓
wait until operation can proceed
```

**Non-blocking:**

```text
read()
  ↓
can't do it now
  ↓
return immediately
```

**Asynchronous I/O:**

Conceptually:

```text
start operation
      ↓
continue doing other work
      ↓
kernel/runtime completes operation
      ↓
completion notification
```

Linux historically relied heavily on readiness-based APIs such as:

```text
select
poll
epoll
```

Newer Linux systems also have **io_uring**, which provides a more general high-performance asynchronous I/O interface.

---

# 13. How everything connects

Imagine Kubernetes starts your application container:

```text
Kubernetes Pod
      ↓
Container runtime
      ↓
Linux process
      │
      ├── namespaces
      │      ├── PID
      │      ├── network
      │      ├── mount
      │      └── ...
      │
      ├── cgroups v2
      │      ├── CPU
      │      ├── memory
      │      └── I/O
      │
      └── application
             │
             ├── threads/processes
             │
             └── epoll/io_uring
                    ↓
              concurrent I/O
```

Then Kubernetes says:

```yaml
limits:
  memory: 1Gi
  cpu: "1"
```

Those eventually become Linux-level resource controls.

Your application may internally have:

```text
1 process
4 threads
10,000 sockets
```

and use non-blocking I/O + `epoll` to handle those connections concurrently without needing 10,000 threads.

---

## The 30-second interview summary

Memorize this relationship:

> **A Linux process has its own virtual address space and one or more threads scheduled by the kernel. Concurrency allows multiple tasks to make progress, while parallelism means executing simultaneously on multiple CPU cores. Linux namespaces isolate what processes can see, such as PIDs, networking and mounts, while cgroups control how many resources those processes can consume, such as CPU and memory. cgroups v2 improves on v1 by providing a unified hierarchy. Containers combine namespaces, cgroups and other Linux isolation mechanisms. For high network concurrency, applications commonly use non-blocking I/O with event mechanisms such as epoll so a small number of threads can efficiently handle large numbers of connections.**