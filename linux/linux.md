# Linux Interview Cheatsheet — Deep Dive

Below is a cheat sheet you can revise before the interview.


## 1. Linux Process Model

### Process vs Thread

**Process**
- Independent execution unit.
- Has its own virtual address space.
- Own PID, file descriptors, memory mappings, etc.
- More expensive to create/context-switch than a thread.

**Thread**
- Execution unit inside a process.
- Threads share the process's memory/address space and many resources.
- Each thread has its own stack, registers and scheduling state.

**Interview answer:**

> A process is an isolated execution environment with its own virtual address space, while threads are lightweight execution units within a process that share memory and resources. This makes threads cheaper for concurrent work but introduces synchronization and locking concerns.

---

## 2. PID / PPID

```bash
ps -ef
```

Example:

```text
UID    PID   PPID  CMD
root   1000     1  nginx
root   1050  1000  nginx worker
```

- **PID** = Process ID
- **PPID** = Parent Process ID

Useful:

```bash
ps -o pid,ppid,stat,cmd -p <PID>
```

To see process hierarchy:

```bash
pstree -p
```

---

# 3. Process States

Linux processes commonly appear as:

| State | Meaning |
|---|---|
| `R` | Running/runnable |
| `S` | Interruptible sleep |
| `D` | Uninterruptible sleep, usually I/O |
| `T` | Stopped |
| `Z` | Zombie |

### Important: `D` state

If you see:

```text
process D
```

don't immediately kill it.

It may be waiting for:

- disk I/O
- NFS
- block device
- storage subsystem

This can explain:

> CPU is low but application is extremely slow.

---

# 4. Zombie vs Orphan

### Zombie

A child process has **finished**, but its parent hasn't collected its exit status.

```text
Parent
  |
  +--- Zombie
```

Zombie doesn't really consume CPU/memory like a running process, but its PID/process-table entry remains.

Check:

```bash
ps aux | grep Z
```

### Orphan

Parent dies while child is still running.

The orphan gets adopted by another process, traditionally PID 1/systemd.

```text
Parent dies
     ↓
Child becomes orphan
     ↓
adopted by PID 1
```

**Interview trap:**

> Zombie ≠ orphan.

Zombie is **already dead**.  
Orphan is **still running**.

---

# 5. Signals

Signals are asynchronous notifications sent to processes.

Common ones:

```text
SIGTERM = 15
SIGKILL = 9
SIGHUP  = 1
SIGINT  = 2
SIGSTOP = 19
```

### SIGTERM

```bash
kill -15 <PID>
```

Means:

> Please terminate gracefully.

Application can handle it and:

- finish current request
- close connections
- flush buffers
- release resources
- exit cleanly

### SIGKILL

```bash
kill -9 <PID>
```

Means:

> Stop immediately.

The application **cannot catch or handle SIGKILL**.

Therefore:

> Always prefer SIGTERM first; use SIGKILL only when graceful termination fails.

---

# 6. fork() vs exec()

This is a classic Linux question.

### fork()

Creates a new process based on the calling process.

```text
Parent
  |
 fork()
  |
  +---- Child
```

### exec()

Replaces the current process image with another program.

Conceptually:

```text
Process
   |
 exec()
   ↓
Different program
```

A common shell flow is:

```text
shell
 ↓
fork()
 ↓
child
 ↓
exec()
 ↓
command
```

For example:

```bash
ls
```

The shell creates a child and that child executes `ls`.

---

# 7. Context Switching

The CPU switches from one execution context to another.

Example:

```text
Process A
   ↓
context switch
   ↓
Process B
   ↓
context switch
   ↓
Process A
```

Context switching requires saving/restoring CPU state.

Too many context switches can indicate:

- excessive threads
- high concurrency
- synchronization problems
- scheduler pressure

Check:

```bash
vmstat 1
```

Look at:

```text
cs
```

for context switches.

---

# 8. CPU Load ≠ CPU Utilization

This is extremely important.

### CPU utilization

How much CPU capacity is currently being used.

Example:

```text
CPU utilization = 40%
```

### Load average

Represents the number of tasks that are:

- runnable
- or waiting in uninterruptible state

Example:

```text
load average: 30, 28, 25
```

on a machine with:

```text
32 CPUs
```

may be perfectly acceptable.

But:

```text
load average: 30
```

on:

```text
4 CPUs
```

is concerning.

### Interview statement

> CPU utilization tells me how busy the CPUs are, while load average tells me how many tasks are competing for CPU or stuck in uninterruptible wait.

---

# 9. The 32 CPU / 40% CPU Question

> A Linux server has 32 CPUs but CPU utilization is only 40%. Requests are extremely slow. Why?

Don't say:

> CPU is fine, so it must be the network.

Think systematically.

### Possibilities

**1. I/O wait**

Application may be waiting on:

```text
Disk
NFS
EBS
Database
```

Check:

```bash
iostat -xz 1
```

Look at:

```text
await
%util
```

---

**2. Network latency**

Application may be waiting for:

```text
Database
API
Service
DNS
Load balancer
```

Check:

```bash
ss -s
```

and:

```bash
tcpdump
```

---

**3. Lock contention**

You could have:

```text
32 CPUs
10 threads
9 waiting on one lock
```

CPU won't necessarily be saturated.

---

**4. Thread pool exhaustion**

Example:

```text
Application thread pool = 100

100 threads waiting for DB connections
DB pool = 20
```

CPU can be low while requests queue.

---

**5. Memory pressure**

Memory pressure can cause:

```text
page faults
reclaim
swap
```

Check:

```bash
free -h
vmstat 1
```

---

**6. Disk latency**

Even with low throughput, high latency can make applications slow.

---

### Strong interview answer

> I wouldn't conclude that the server is healthy just because CPU utilization is 40%. I'd check CPU run queue and load, I/O wait and disk latency, memory pressure and swap, network latency, thread or connection-pool saturation, and application-level lock contention. I'd correlate those with application latency metrics to identify where the request is actually waiting.

That's a **much stronger Senior-level answer**.

---

# 10. CPU Steal

Very important in cloud environments.

**CPU steal time** means the VM wanted CPU but the hypervisor was servicing another workload.

Example:

```text
VM
 ↓
Hypervisor
 ↓
Physical CPU
```

If:

```text
%steal = high
```

the VM isn't getting all the CPU time it expects.

Useful:

```bash
top
```

Look for:

```text
st
```

or:

```bash
mpstat
```

### Interview answer

> High CPU steal can indicate contention for physical CPU resources at the hypervisor level, particularly in virtualized environments.

---

# 11. Linux Memory Model

Think:

```text
Application
     ↓
Virtual Memory
     ↓
Virtual Pages
     ↓
Physical Pages
     ↓
RAM
```

Every process sees a **virtual address space**, not raw physical memory.

The kernel maps virtual pages to physical memory.

---

# 12. Virtual Memory

Suppose:

```text
Application requests:

Virtual address 0x1234
```

The MMU/page tables translate it to something like:

```text
Physical address 0xABCD
```

This provides:

- process isolation
- memory protection
- virtual address spaces
- ability to use more memory than immediately available RAM through paging/swap

---

# 13. Pages

Linux manages memory largely in fixed-size units called **pages**.

Typical page size:

```text
4 KB
```

Conceptually:

```text
RAM
+-----+
|page |
+-----+
|page |
+-----+
|page |
+-----+
```

Processes allocate memory, but the kernel manages physical memory using pages.

---

# 14. Page Cache

One of the most commonly misunderstood Linux concepts.

When an application reads a file:

```text
Application
     ↓
Filesystem
     ↓
Page Cache
     ↓
Disk
```

Linux keeps recently accessed filesystem data in RAM.

So:

```bash
free -h
```

may show little "free" memory.

That doesn't necessarily mean memory is exhausted.

Linux intentionally uses RAM for cache.

---

# 15. Swap

Swap provides disk-backed memory when RAM is under pressure.

Conceptually:

```text
RAM
 ↓
Memory pressure
 ↓
Pages moved to swap
 ↓
Disk
```

But swap is much slower than RAM.

High swap activity can cause severe performance degradation.

Check:

```bash
free -h
vmstat 1
```

Look at:

```text
si
so
```

---

# 16. OOM Killer

When Linux cannot satisfy memory allocations under memory pressure, the kernel may invoke the **OOM killer**.

It selects a process to terminate based on factors such as its memory usage and OOM score.

Check:

```bash
dmesg | grep -i oom
```

or:

```bash
journalctl -k | grep -i oom
```

### Kubernetes connection

This becomes particularly important with containers.

```text
Container
   ↓
cgroup memory limit
   ↓
memory exceeds limit
   ↓
OOMKilled
```

Check:

```bash
kubectl describe pod <pod>
```

You may see:

```text
Reason: OOMKilled
Exit Code: 137
```

---

# 17. RSS

**RSS = Resident Set Size**

It represents the amount of a process's memory currently resident in physical RAM.

Check:

```bash
ps aux
```

or:

```bash
ps -o pid,rss,vsz,cmd -p <PID>
```

Important distinction:

```text
VSZ = virtual memory size
RSS = physical memory currently resident
```

Don't confuse VSZ with actual RAM consumption.

---

# 18. Memory Leak

A memory leak occurs when an application continues allocating memory without releasing memory it no longer needs.

Example:

```text
Hour 1 → 1 GB
Hour 2 → 2 GB
Hour 3 → 4 GB
Hour 4 → 8 GB
```

Eventually:

```text
OOM
```

For containers:

```text
Application
   ↓
memory grows
   ↓
cgroup limit
   ↓
OOMKilled
```

---

# 19. cgroups

This is **very important for Kubernetes interviews**.

Linux cgroups provide resource controls for groups of processes.

They can control:

- CPU
- memory
- PIDs
- I/O

Kubernetes uses cgroups underneath the container runtime.

Conceptually:

```text
Kubernetes Pod
      ↓
Container
      ↓
container runtime
      ↓
Linux cgroup
      ↓
Kernel
```

So when you specify:

```yaml
resources:
  limits:
    memory: "1Gi"
```

you're ultimately relying on Linux resource isolation mechanisms.

---

# 20. Storage Stack

Remember this architecture:

```text
Application
     ↓
System Calls
     ↓
VFS
     ↓
Filesystem
     ↓
Block Layer
     ↓
Block Device
     ↓
Disk / EBS / NVMe
```

### VFS

**Virtual File System**

Provides a common interface regardless of whether you're using:

```text
ext4
xfs
NFS
etc.
```

---

# 21. Inode

An inode contains filesystem metadata about a file.

Think:

```text
Filename
   ↓
Directory entry
   ↓
inode
   ↓
file metadata + data blocks
```

Inode contains things like:

- permissions
- owner
- timestamps
- file size
- pointers to data blocks

### Important production issue

You can have:

```text
Disk space available = YES
```

but:

```text
Inodes exhausted = YES
```

Then you may be unable to create new files.

Check:

```bash
df -h
df -i
```

This is a good interview example.

---

# 22. File Descriptors

A file descriptor is an integer handle representing an open resource.

Examples:

```text
file
socket
pipe
device
```

Common:

```text
0 = stdin
1 = stdout
2 = stderr
```

Example:

```text
Process
 ├── FD 0
 ├── FD 1
 ├── FD 2
 ├── FD 3 → file
 ├── FD 4 → socket
 └── FD 5 → socket
```

---

# 23. File Descriptor Exhaustion — Great Production Example

Suppose:

```text
Application
     ↓
opens socket
     ↓
doesn't close it
     ↓
FD count continuously increases
     ↓
ulimit reached
```

Application starts failing:

```text
Too many open files
```

Check:

```bash
ulimit -n
```

Process-specific:

```bash
cat /proc/<PID>/limits
```

Open files:

```bash
lsof -p <PID>
```

Count:

```bash
ls /proc/<PID>/fd | wc -l
```

### Interview answer

> I once encountered a scenario where an application became unable to establish new connections even though CPU and memory were healthy. I checked the application logs and found "too many open files." I inspected `/proc/<pid>/limits` and the process's file descriptors with `lsof`, confirmed FD exhaustion, and then traced it back to connections not being closed properly. The immediate mitigation was to restart the affected workload and adjust limits where appropriate, but the permanent fix was correcting the application's resource lifecycle.

That's the kind of answer that demonstrates **actual Linux troubleshooting**.

---

# 24. `lsof`

Think:

> **List Open Files**

But "files" in Linux includes:

```text
regular files
directories
sockets
pipes
devices
```

Examples:

```bash
lsof -p <PID>
```

Find what's using port 8080:

```bash
lsof -i :8080
```

Find deleted files still held open:

```bash
lsof | grep deleted
```

That last one is a **great production troubleshooting trick**.

---

# 25. Deleted File Still Consuming Disk

Interesting Linux scenario:

```text
Application opens huge.log
        ↓
logrotate deletes huge.log
        ↓
Application still has FD open
        ↓
Disk space remains consumed
```

`df` says:

```text
Disk 95% full
```

but:

```bash
du -sh /*
```

doesn't explain the usage.

Check:

```bash
lsof | grep deleted
```

You may find:

```text
java ... /var/log/app.log (deleted)
```

Restarting/closing the process releases the underlying space.

---

# 26. `strace`

`strace` lets you observe system calls made by a process.

Example:

```bash
strace -p <PID>
```

You may see:

```text
read(...)
write(...)
openat(...)
connect(...)
poll(...)
futex(...)
```

This helps answer:

> What is the process actually waiting on?

### Example

Application is slow.

`strace` shows:

```text
connect(...)
<waiting>
connect(...)
<waiting>
```

Potentially network-related.

Or:

```text
futex(...)
futex(...)
futex(...)
```

could point toward synchronization/locking contention.

---

# 27. `ss`

Modern tool for inspecting sockets.

```bash
ss -tulpn
```

Shows listening TCP/UDP ports.

Useful:

```bash
ss -s
```

For established connections:

```bash
ss -tan
```

For a particular port:

```bash
ss -tan '( sport = :443 )'
```

---

# 28. `tcpdump`

Packet-level troubleshooting.

Example:

```bash
tcpdump -i eth0 port 443
```

You can investigate:

```text
DNS
TCP handshake
retransmissions
packet loss
connection resets
unexpected traffic
```

### TCP troubleshooting mental model

```text
Client
  |
SYN
  ↓
Server
  |
SYN-ACK
  ↓
Client
  |
ACK
  ↓
Connection established
```

If SYNs leave but SYN-ACK doesn't return:

Think:

- firewall
- security group
- routing
- server not listening
- NetworkPolicy
- network path

---

# 29. `iostat`

Disk/storage troubleshooting.

```bash
iostat -xz 1
```

Important metrics:

### `%util`

How busy the device is.

### `await`

Average I/O wait latency.

### `r/s`, `w/s`

Read/write operations per second.

### `rkB/s`, `wkB/s`

Read/write throughput.

---

# 30. IOPS vs Throughput vs Latency

Very important for cloud/SRE interviews.

### IOPS

Number of I/O operations per second.

```text
100,000 operations/sec
```

### Throughput

Amount of data transferred per second.

```text
500 MB/sec
```

### Latency

Time required for an individual operation.

```text
2 ms
```

A workload can have:

```text
Low throughput
High latency
```

and still be extremely slow.

---

# 31. `vmstat`

Excellent overall system-health tool.

```bash
vmstat 1
```

Useful things:

```text
r   → runnable processes
si  → swap in
so  → swap out
wa  → I/O wait
cs  → context switches
```

If you see:

```text
wa = high
```

investigate storage/I/O.

If:

```text
si/so = high
```

investigate memory pressure/swap.

If:

```text
r = very high
```

investigate CPU contention.

---

# 32. `sar`

Historical system performance.

Examples:

```bash
sar -u
sar -r
sar -n DEV
sar -d
```

Useful when:

> "The server was slow at 2 AM, but it's fine now."

Instead of saying:

> "I can't reproduce it."

You can inspect historical metrics.

---

# 33. `journalctl`

Systemd logs.

```bash
journalctl
```

Specific service:

```bash
journalctl -u nginx
```

Follow logs:

```bash
journalctl -u nginx -f
```

Since boot:

```bash
journalctl -b
```

Kernel messages:

```bash
journalctl -k
```

---

# 34. `dmesg`

Kernel-level messages.

```bash
dmesg
```

Useful for:

- OOM
- disk errors
- driver problems
- kernel issues
- hardware problems
- filesystem errors

Example:

```bash
dmesg | grep -i oom
```

---

# 35. `perf`

For deeper performance profiling.

```bash
perf top
```

Can help identify where CPU time is being spent.

Conceptually:

```text
Application
    ↓
perf
    ↓
CPU hotspots
    ↓
function/system-call analysis
```

You probably won't need to use `perf` every day, but knowing **when to use it** demonstrates senior-level Linux knowledge.

---

# 36. Production Troubleshooting Flow

This is what I would memorize.

When somebody says:

> "The Linux server/application is slow."

Don't randomly run commands.

Think:

```text
1. Is the problem CPU?
        ↓
2. Is it memory?
        ↓
3. Is it disk/I/O?
        ↓
4. Is it network?
        ↓
5. Is it process/thread contention?
        ↓
6. Is it resource exhaustion?
        ↓
7. Is it application-level?
```

### Commands

```bash
uptime
top
vmstat 1
free -h
iostat -xz 1
ss -s
df -h
df -i
lsof
dmesg
journalctl
```

Then drill down using:

```bash
strace
tcpdump
perf
```

---

# 37. A Strong Real-Project Scenario

### Scenario

> Production API suddenly became slow.

Initial metrics:

```text
CPU = 35%
Memory = 55%
Disk = 60%
```

A junior engineer might say:

> Infrastructure looks healthy.

Instead:

### Step 1 — Load

```bash
uptime
```

Load is elevated relative to CPU count.

### Step 2 — vmstat

```bash
vmstat 1
```

Shows:

```text
wa = high
```

So investigate I/O.

### Step 3 — iostat

```bash
iostat -xz 1
```

Shows:

```text
await = high
```

Disk latency is high.

### Step 4 — Application

Application is heavily dependent on database/storage operations.

### Step 5 — Correlation

Application latency increased at exactly the same time as storage latency.

### Root cause

Not CPU.

Not memory.

**Storage I/O latency caused request latency.**

### Strong interview narration

> I started with the four golden signals—latency, traffic, errors and saturation—and then correlated them with host-level metrics. CPU and memory weren't saturated, so I checked load, vmstat and iostat. High I/O wait and disk latency indicated a storage bottleneck. We then correlated that with application and database metrics, identified the affected storage workload, mitigated the immediate impact and subsequently addressed the underlying I/O pattern.

---

# 38. Linux → Kubernetes Connection

For your interviews, make this connection explicitly.

```text
Kubernetes
     ↓
Container
     ↓
Container Runtime
     ↓
Linux Kernel
     ├── namespaces
     ├── cgroups
     ├── processes
     ├── networking
     ├── filesystem
     └── storage
```

### Kubernetes is heavily dependent on Linux primitives.

| Kubernetes concept | Linux concept |
|---|---|
| Container isolation | Namespaces |
| CPU limits | cgroups |
| Memory limits | cgroups |
| Process isolation | PID namespace |
| Network isolation | Network namespace |
| Container filesystem | Overlay filesystem |
| Container networking | Linux networking |
| Persistent storage | Block/filesystem layer |

This connection is **very useful for a Principal/Senior SRE interview**.

---

# 39. The 10 Commands I'd Prioritize

Don't memorize 100 commands.

Master these:

```bash
top
ps
vmstat
iostat
free
ss
lsof
strace
tcpdump
journalctl
```

And understand what question each answers:

```text
top       → What's consuming resources?
ps        → What processes exist?
vmstat    → What's happening to CPU/memory/I/O?
iostat    → Is storage slow?
free      → What's happening with memory?
ss        → What's happening with sockets?
lsof      → What resources are processes holding?
strace    → What is the process actually doing?
tcpdump   → What is happening on the network?
journalctl → What did the system/service report?
```

## Final interview mindset

When troubleshooting Linux, think:

**CPU → Memory → I/O → Network → Processes → Resources → Application**

And don't just say:

> "I'll run `top`."

Say:

> **"First I'll establish whether the bottleneck is CPU, memory, I/O, network, process contention, or resource exhaustion, then drill down using the appropriate kernel-level tools and correlate the findings with application metrics."**
