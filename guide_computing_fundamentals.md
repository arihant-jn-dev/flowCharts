# Computing Fundamentals — CPU, Cores, Threads, Processes & More

**Document Type:** Engineering Education / Reference
**Author:** LeadCore Team
**Date:** April 2026
**Companion to:** `NODEJS_INTERNALS_DEEP_DIVE.md`

---

## TL;DR — What This Document Is About

Before you can truly understand why Node.js behaves the way it does, you need to understand the hardware and OS concepts underneath it. Terms like "CPU core," "thread," "process," "context switch," "I/O," and "memory" get thrown around constantly — but most engineers only have a vague idea of what they actually mean.

This document explains every fundamental term from scratch, using simple physical analogies and then connecting each concept directly to what you see in your Node.js or PHP server.

Read this first. Then the Node.js internals doc will make complete sense.

---

## Table of Contents

1. [The CPU — The Brain of the Computer](#1-the-cpu--the-brain-of-the-computer)
2. [CPU Cores — Multiple Brains in One Chip](#2-cpu-cores--multiple-brains-in-one-chip)
3. [Clock Speed — How Fast the Brain Thinks](#3-clock-speed--how-fast-the-brain-thinks)
4. [RAM (Memory) — The Desk You Work On](#4-ram-memory--the-desk-you-work-on)
5. [Storage (Disk / SSD) — The Filing Cabinet](#5-storage-disk--ssd--the-filing-cabinet)
6. [Processes — Independent Programs Running](#6-processes--independent-programs-running)
7. [Threads — Workers Inside a Process](#7-threads--workers-inside-a-process)
8. [Context Switching — The OS Juggles Workers](#8-context-switching--the-os-juggles-workers)
9. [I/O — Input and Output Operations](#9-io--input-and-output-operations)
10. [Blocking vs Non-Blocking — Waiting vs Delegating](#10-blocking-vs-non-blocking--waiting-vs-delegating)
11. [Synchronous vs Asynchronous — Two Ways to Ask](#11-synchronous-vs-asynchronous--two-ways-to-ask)
12. [The Stack and the Heap — Two Types of Memory](#12-the-stack-and-the-heap--two-types-of-memory)
13. [System Calls — How Programs Talk to the OS](#13-system-calls--how-programs-talk-to-the-os)
14. [Network I/O — Talking to Other Machines](#14-network-io--talking-to-other-machines)
15. [CPU-Bound vs I/O-Bound Work](#15-cpu-bound-vs-io-bound-work)
16. [Concurrency vs Parallelism](#16-concurrency-vs-parallelism)
17. [How Everything Connects to Node.js and LeadCore](#17-how-everything-connects-to-nodejs-and-leadcore)

---

## 1. The CPU — The Brain of the Computer

**CPU** stands for **Central Processing Unit**. It is the chip that actually executes instructions — the calculations, comparisons, and logic that make your code run.

### Physical Reality

A CPU is a piece of silicon roughly the size of a postage stamp. Modern CPUs (like what AWS uses in EC2 / Fargate) have billions of transistors etched into them — tiny electrical switches that turn on and off billions of times per second to perform computations.

```
WHAT THE CPU DOES:

Your code (PHP, Node.js, Go) is compiled into machine instructions.
Each instruction is something like:
  - ADD two numbers
  - COMPARE two values
  - COPY this memory to that memory
  - JUMP to this instruction if condition is true

The CPU executes these one by one, billions per second.
```

### What the CPU Is NOT Responsible For

The CPU does not:
- Wait for a network response (that's the network card + OS)
- Wait for a disk read (that's the disk controller + OS)
- Wait for a Redis response (that's the network + OS)

When your code asks for a Redis value, the CPU sends the request and could immediately do other work. The question is: **does your programming language and runtime let it?** This is exactly the Node.js vs PHP-FPM difference.

---

## 2. CPU Cores — Multiple Brains in One Chip

A **core** is a complete, independent execution unit inside the CPU. Each core can execute one instruction stream at a time, completely independently of the others.

### Physical Analogy

Think of the CPU as a **factory building**. Each core is a **separate worker station** inside that factory. They share the same building (chip) and some shared resources (cache), but each can work on a completely different task simultaneously.

```
MODERN CPU CHIP (e.g., AWS Fargate vCPU)

┌─────────────────────────────────────────────────────┐
│                  CPU Chip                           │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  Core 1  │  │  Core 2  │  │  Core 3  │  ...     │
│  │          │  │          │  │          │          │
│  │ executes │  │ executes │  │ executes │          │
│  │ thread A │  │ thread B │  │ thread C │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │          Shared L3 Cache                    │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### How Many Cores Does Your Server Have?

```bash
# On a Linux server (like your ECS task)
nproc           # outputs: 2  (for a 2-vCPU Fargate task)
lscpu           # detailed info

# On your Mac
sysctl -n hw.ncpu   # outputs: 10 (M-series has 10 cores)
```

### What "vCPU" Means in AWS

AWS uses **vCPU** (virtual CPU), which is typically one hardware thread of a physical core. When you allocate `0.5 vCPU` in Fargate, you get access to half of one physical core's time. `2 vCPU` gives you two full independent execution units.

For LeadCore on Fargate (`0.5 vCPU`):
- Your Go service gets access to one physical core for ~50% of the time
- One Node.js process running on `0.5 vCPU` has only half a core to work with

---

## 3. Clock Speed — How Fast the Brain Thinks

**Clock speed** (measured in GHz — Gigahertz) tells you how many instruction cycles a CPU can execute per second.

```
1 GHz  = 1,000,000,000 cycles per second
3 GHz  = 3,000,000,000 cycles per second  (typical modern server)
3.5 GHz = 3,500,000,000 cycles per second

One simple instruction (like adding two numbers) takes 1-4 cycles.
At 3 GHz, a core can execute roughly 1-3 BILLION simple operations per second.
```

### Why This Matters for LeadCore

Evaluating one targeting rule (age check, state check) takes a handful of CPU instructions — maybe 50 cycles. At 3 GHz, that's 0.000016ms (16 nanoseconds).

Evaluating 200 offers × ~100 instructions each = 20,000 instructions.
Time: 20,000 / 3,000,000,000 = 0.0067ms = **6.7 microseconds**.

This is why CPU work for targeting is measured in microseconds — CPUs are extraordinarily fast at arithmetic and comparisons. The bottleneck is never "can the CPU do the math" — it's "is the CPU free to do the math, or is it waiting for I/O?"

---

## 4. RAM (Memory) — The Desk You Work On

**RAM** (Random Access Memory) is where your running programs live. It is fast, temporary storage that the CPU can read from and write to almost instantly.

### Physical Analogy

```
ANALOGY: An Office Worker

CPU     = The worker's brain (does the thinking)
RAM     = The worker's desk (what's currently spread out and accessible)
Disk    = The filing cabinet in the hallway (slower to access, but permanent)
Cache   = The worker's notepad on the desk (tiny but instant to read)
```

### Access Speed Comparison

```
Storage type        Access time     Analogy
──────────────────────────────────────────────────────
CPU L1 Cache        0.5 ns          Remembering from your brain
CPU L2 Cache        5 ns            Checking your notepad
CPU L3 Cache        40 ns           Looking at your desk
RAM                 100 ns          Reaching to your filing tray
SSD (local)         50,000 ns       Walking to the filing cabinet
Network (Redis)     2,000,000 ns    Calling someone in another office
Network (MySQL)     5,000,000 ns    Calling someone in another building
Network (MAX API)   150,000,000 ns  Calling someone in another country

ns = nanosecond = 1/1,000,000,000 of a second
```

This is why **in-memory caching** (storing offer rules in Node.js memory) is so powerful. Reading from RAM (100ns) is **20,000× faster** than asking Redis over the network (2,000,000ns = 2ms).

### RAM in Your LeadCore Service

```
LeadCore Node.js process RAM usage:

V8 JavaScript engine:        ~30MB  (runtime overhead)
node_modules:                ~30MB  (libraries loaded)
Offer cache (200 offers):    ~5MB   (your targeting rules in RAM)
Worker thread runtimes:      ~20MB  (4 workers × 5MB each)
Per-request objects:         ~10KB  × concurrent requests
──────────────────────────────────────────────
Total:                       ~90MB  (typical)
```

ECS Fargate task allocation: 512MB. You have ~420MB headroom.

---

## 5. Storage (Disk / SSD) — The Filing Cabinet

**Disk storage** (HDD or SSD) holds data permanently — it survives reboots and process crashes. RAM is lost when power goes off; disk is not.

### Access Speed

- **SSD (local):** 50-100 microseconds per read
- **HDD (spinning disk):** 5,000-10,000 microseconds per read
- **Network storage (EBS on AWS):** 1,000-5,000 microseconds per read

### Relevance to LeadCore

LeadCore itself doesn't read from disk much. The important storage operations are network-based:
- **Redis (ElastiCache):** lives on a different machine → network I/O (~2ms)
- **MySQL (Aurora):** lives on a different machine → network I/O (~5ms)
- **Kinesis:** lives in AWS → network I/O (~10ms)

This is why every database/cache call in your server is an **I/O operation**, not a local disk read.

---

## 6. Processes — Independent Programs Running

A **process** is a running instance of a program. When you start your Node.js server, the OS creates a process for it.

### What a Process Contains

```
PROCESS (e.g., your Node.js server)

┌─────────────────────────────────────────────────────┐
│                    Process                          │
│                                                     │
│  Code Segment:   your compiled JavaScript/bytecode  │
│  Data Segment:   global variables                   │
│  Heap:           dynamically allocated memory       │
│  Stack:          function call frames               │
│  File Handles:   open sockets, files, pipes         │
│  PID:            unique process ID (e.g., 1247)     │
│  Environment:    env variables (PORT, REDIS_URL)    │
└─────────────────────────────────────────────────────┘
```

### Process Isolation

Processes are **completely isolated** from each other. Process A cannot read Process B's memory. If Process A crashes, Process B is unaffected.

```
PROCESS ISOLATION:

PHP-FPM Worker 1 (PID 1001):
  handles request A, crashes → only request A fails
  Process 1002, 1003, 1004 are unaffected

PHP-FPM Worker 2 (PID 1002):
  still serving request B normally

VS

Node.js Server (PID 1050):
  single process handles all requests
  if it crashes → ALL in-flight requests fail
  (this is why error handling and ECS auto-restart matters)
```

### Processes on Your AWS Setup

```
ECS Fargate Task (a container = isolated environment):
  - 1 Node.js process (PID 1)
  - libuv background threads (PIDs 2-8)
  - Health check process (PID 9)

If the main process (PID 1) crashes:
  ECS detects health check failure
  ECS stops the task and starts a new one
  ALB routes traffic to other healthy tasks during this time
```

---

## 7. Threads — Workers Inside a Process

A **thread** is an independent sequence of instructions running **inside a process**. A process can have one thread (single-threaded) or many threads (multi-threaded).

### Thread vs Process Differences

```
PROCESS                          THREAD
─────────────────────────────────────────────────────────
Has its own memory space          Shares memory with other threads in same process
Isolated from other processes     Can read/write same variables as other threads
Expensive to create (~1ms)        Cheap to create (~10 microseconds)
Expensive to communicate          Cheap to communicate (shared memory)
OS kills process = all threads    Thread dies = process continues (usually)
  inside die too
Separate address space            Same address space
```

### Thread Analogy — Office Team

```
PROCESS = A Department (e.g., Engineering)
  Has its own office space (memory)
  Has its own budget (resources)
  Has its own employees (threads)

THREADS = Individual Engineers in the Department
  Share the same office (memory)
  Can talk to each other instantly (shared variables)
  Each can work on a different task simultaneously
  If one engineer has a breakdown (crash), it can affect the whole department
```

### How Threads Use CPU Cores

```
4-Core CPU + 4-Thread program:

Core 1 ──► Thread 1 (runs simultaneously)
Core 2 ──► Thread 2 (runs simultaneously)
Core 3 ──► Thread 3 (runs simultaneously)
Core 4 ──► Thread 4 (runs simultaneously)

All 4 threads run at the exact same instant = TRUE PARALLELISM

4-Core CPU + 100-Thread program:

Core 1 ──► Thread 1, then 2, then 3... (rapid switching)
Core 2 ──► Thread 4, then 5, then 6... (rapid switching)
Core 3 ──► Thread 7, then 8, then 9... (rapid switching)
Core 4 ──► Thread 10, then 11...       (rapid switching)

Not enough cores for all threads → OS rapidly switches (context switch)
```

### Thread Safety Problem

Because threads share memory, two threads can try to change the same variable at the same time — causing a **race condition**:

```
Thread 1:  reads budget = 100
Thread 2:  reads budget = 100      ← reads SAME value before Thread 1 writes
Thread 1:  subtracts 10, writes budget = 90
Thread 2:  subtracts 10, writes budget = 90  ← WRONG! Should be 80!

Result: budget decremented twice but only reduced by 10, not 20.
Money leaked.
```

This is exactly why we use **Redis Lua scripts** for budget decrement — the Lua script runs atomically on Redis (like a single thread), so no two operations can interleave.

### Node.js and Threads

```
Node.js process thread structure:

Main Thread:
  ├── V8 JavaScript Engine (your app code)
  ├── Event Loop (coordinates I/O callbacks)
  └── libuv Thread Pool (4 background OS threads for DNS, file I/O)
      ├── OS Thread 1
      ├── OS Thread 2
      ├── OS Thread 3
      └── OS Thread 4

Worker Threads (if you add them):
  ├── Worker 1 (runs JavaScript in separate thread — targeting eval)
  └── Worker 2 (runs JavaScript in separate thread — targeting eval)
```

The key point: **your application JavaScript code only runs on the Main Thread**. libuv background threads handle raw OS-level I/O (not your business logic). Worker Threads let you run your own code on other threads.

---

## 8. Context Switching — The OS Juggles Workers

**Context switching** is when the OS pauses one thread and resumes another on the same CPU core.

### Why Context Switching Happens

A CPU core can only execute one thread at a time. If you have 100 threads and 2 cores, the OS rapidly switches between threads, giving each a tiny time slice (~1-10ms). This creates the **illusion** of parallelism for more threads than cores.

### What Happens During a Context Switch

```
Context Switch: Thread A → Thread B on Core 1

1. OS SAVES Thread A's state:
   - All CPU register values (where A was in its calculation)
   - Program counter (which instruction A was on)
   - Stack pointer
   → Saved to Thread A's "context" in RAM

2. OS LOADS Thread B's state:
   - Restores CPU registers for Thread B
   - Restores program counter (resumes where B left off)
   - Restores stack pointer
   → Loaded from Thread B's "context" in RAM

3. Core 1 continues executing Thread B

Cost: ~1-10 microseconds per context switch
```

### Why This Matters for PHP-FPM (Multi-Process)

PHP-FPM creates one process per request. With 100 concurrent requests and 100 PHP processes:

```
100 PHP processes on 4 cores:
  4 cores × 100 processes = 25 processes per core
  Each process gets a time slice every few milliseconds
  OS performs 100s of context switches per second

Context switch cost: 5 microseconds × 100 switches/sec = 500 microseconds/sec overhead
Plus: 100 processes × 30MB RAM = 3GB just for PHP workers
```

### Why Node.js Avoids Most of This Cost

Node.js has **one main thread** for JavaScript. The OS only context-switches that one thread with others (the OS, libuv threads, etc.). For CPU work, Node.js needs no context switching — it just runs. For I/O, the OS does the waiting, not the JavaScript thread.

```
Node.js context switch overhead:
  Main thread: minimal switching (mostly the thread is either running or parked on poll)
  Worker threads: switch between workers — but you control how many (keep it ≤ core count)
```

---

## 9. I/O — Input and Output Operations

**I/O** (Input/Output) refers to any operation where your program exchanges data with something outside the CPU and RAM — a disk, a network, a keyboard, a screen.

### Types of I/O Relevant to LeadCore

```
TYPE                  EXAMPLE                         SPEED
─────────────────────────────────────────────────────────────────────
Network I/O           Redis GET, MySQL query          1-20ms
  (same VPC)          Kinesis write
Network I/O           MAX API call                    150-500ms
  (external)
File I/O              Reading config file             0.05-5ms
  (local disk/SSD)
Pipe I/O              Node.js → Worker Thread         0.01ms
  (inter-process)     (via postMessage)
```

### Why I/O Is Slow vs CPU

```
CPU doing arithmetic:    1-4 cycles = ~1 nanosecond
RAM access:              ~100 nanoseconds
Redis GET (network):     ~2,000,000 nanoseconds = 2ms

Redis is 2,000,000× slower than a CPU instruction.

During those 2ms, at 3 GHz, the CPU could have executed:
  3,000,000,000 × 0.002 = 6,000,000 instructions

That is 6 million wasted instructions if the thread just sits and waits.
```

This is why **not wasting the CPU while waiting for I/O** is such a big deal. Non-blocking I/O lets the CPU do useful work (handle other requests) during those 6 million "free" instruction slots.

### The Journey of a Redis GET Request

```
Node.js code:
  redis.get('budget:123')
  ↓
V8 calls ioredis library
  ↓
ioredis calls Node.js C++ binding
  ↓
C++ binding calls libuv
  ↓
libuv performs write() system call: sends "GET budget:123\r\n" to TCP socket
  ↓
OS Kernel: copies data to network card buffer
  ↓
Network Card: encodes as electrical signals, sends over wire
  ↓
  [time passes: 1-3ms, the CPU is FREE during this time]
  ↓
Redis Server: processes command, sends response
  ↓
Network Card: receives electrical signals, decodes, writes to OS buffer
  ↓
OS Kernel: notifies libuv via epoll "data available on socket"
  ↓
libuv: queues the callback in Node.js event queue
  ↓
Event Loop: picks up callback, passes result to ioredis
  ↓
ioredis: resolves the Promise
  ↓
Your code: const value = await redis.get(...)  ← resumes here
```

Every step above the "time passes" line and below it is essentially instant (microseconds). The 2ms is all spent on the wire.

---

## 10. Blocking vs Non-Blocking — Waiting vs Delegating

This is the single most important concept for understanding Node.js performance.

### Blocking I/O — Your Thread Waits

```
BLOCKING I/O (what PHP does for most operations):

Thread Timeline:
─────────────────────────────────────────────────────────────────────
Thread 1: [CODE]──► sends Redis request ──► WAITING ──► WAITING ──► [CODE resumes]
                                            (1ms)       (1ms)
                    [Thread is frozen, doing nothing, consuming a thread slot]
```

```php
// PHP — synchronous, blocking by default
$value = $redis->get('budget:123');
// Thread literally stops here and waits ~2ms
// No other request can use this thread during those 2ms
echo $value; // runs after Redis responds
```

### Non-Blocking I/O — The OS Waits, Your Thread Continues

```
NON-BLOCKING I/O (what Node.js does):

Thread Timeline:
─────────────────────────────────────────────────────────────────────
Thread 1: [CODE]──► sends Redis request ──► [MORE CODE] ──► [Redis arrives] ──► [RESUME]
                                            (handles          (callback queued)
                                             other requests)
```

```javascript
// Node.js — asynchronous, non-blocking
const value = await redis.get('budget:123');
// "await" suspends THIS function — but NOT the thread
// Thread handles other requests while Redis fetches data
// This function resumes here when Redis responds
console.log(value);
```

### The Waiter Analogy

```
BLOCKING (synchronous) — Bad Waiter:
  Waiter takes Order A → goes to kitchen → stands there staring at the cook
  → cook finishes → brings food to Table A
  → then takes Order B → goes to kitchen → stands there staring...

  One waiter can only serve one table at a time.
  10 tables need 10 waiters (expensive, a lot of people standing around).

NON-BLOCKING (asynchronous) — Good Waiter:
  Waiter takes Order A → goes to kitchen → gives ticket → goes back to floor
  Waiter takes Order B → gives ticket → goes back to floor
  Waiter takes Order C → gives ticket → goes back to floor
  Kitchen calls "Order A ready!" → Waiter picks it up → delivers
  Kitchen calls "Order B ready!" → Waiter picks it up → delivers

  One waiter serves all tables. Efficient, no waste.
```

The "kitchen" is the OS/network. The "waiter" is the Event Loop thread.

---

## 11. Synchronous vs Asynchronous — Two Ways to Ask

**Synchronous** = "Wait for the answer before continuing." Everything happens in order, one after another.

**Asynchronous** = "Ask the question, do other things, come back for the answer when it's ready."

### Synchronous Code Flow

```
Synchronous execution (sequential, each step waits):

Step 1: Fetch budget from Redis    [waits 2ms]
Step 2: Fetch rules from MySQL     [waits 8ms]
Step 3: Calculate targeting        [waits 4ms CPU]
Step 4: Send response

Total time: 2 + 8 + 4 = 14ms
```

### Asynchronous Code Flow

```
Asynchronous execution (concurrent for independent I/O):

t=0ms:   Start Redis fetch (budget)
t=0ms:   Start MySQL fetch (rules)  ← both start immediately
         [waiting for both, doing other things]

t=2ms:   Redis responds → budget ready
t=8ms:   MySQL responds → rules ready

t=8ms:   Both available → Calculate targeting [4ms CPU]
t=12ms:  Send response

Total time: max(2, 8) + 4 = 12ms  ← 14% faster, with zero extra hardware
```

### The Difference in Code

```javascript
// SYNCHRONOUS STYLE (sequential — don't do this in Node.js)
async function sequential() {
    const budget = await redis.get('budget:123');   // 2ms
    const rules  = await mysql.query('SELECT...');  // 8ms — waits for redis first
    const user   = await redis.get('user:456');     // 2ms — waits for mysql first
    // Total: 12ms (sequential)
}

// ASYNCHRONOUS STYLE (concurrent — correct approach)
async function concurrent() {
    const [budget, rules, user] = await Promise.all([
        redis.get('budget:123'),    // starts at t=0
        mysql.query('SELECT...'),   // starts at t=0
        redis.get('user:456')       // starts at t=0
    ]);
    // Total: 8ms (slowest of three, they ran together)
}
```

---

## 12. The Stack and the Heap — Two Types of Memory

Every running program uses two distinct regions of RAM: the stack and the heap.

### The Stack — Fast, Ordered, Automatic

The **stack** is a region of memory that grows and shrinks automatically as functions are called and return. It stores:
- Local variables in functions
- Function arguments
- Return addresses (where to go after a function returns)
- Temporary values

```
STACK in action:

function main() {
    const x = 5;           ← x is on the stack
    result = add(x, 3);    ← call add()
}

function add(a, b) {       ← a, b pushed onto stack
    const sum = a + b;     ← sum on stack
    return sum;            ← sum returned, stack frame for add() removed
}

Stack state during add():
  ┌─────────────────┐  ← top of stack (most recent)
  │ add frame       │
  │   a = 5         │
  │   b = 3         │
  │   sum = 8       │
  ├─────────────────┤
  │ main frame      │
  │   x = 5         │
  │   return addr   │
  └─────────────────┘  ← bottom of stack

When add() returns: its frame is popped. Memory instantly reclaimed.
```

**Characteristics:**
- Very fast (just a pointer increment/decrement)
- Fixed size (typically 1-8MB per thread)
- Automatically managed (no garbage collection needed)
- Limited: you can't store large data here

**Stack Overflow:** If too many functions call each other (deep recursion), the stack runs out of space → "stack overflow" error → crash.

### The Heap — Flexible, Dynamic, Managed

The **heap** is a large region of memory for data that:
- Needs to outlive the function that created it
- Has a size unknown at compile time
- Is too large for the stack

```javascript
// Stack: small, known-size, function-local
function processRequest(req) {
    const requestId = 123;          // stack: 8 bytes, gone when function returns
    const statusCode = 200;         // stack: 8 bytes

    // Heap: large, dynamic, shared
    const offer = {                 // heap: object with unknown fields
        id: 'offer_abc',
        targeting: {
            states: ['NY', 'CA'],
            ageRange: [25, 65],
        }
    };
    // `offer` reference is on the stack, but the actual object is in the heap
    
    return offer; // offer survives this function → stays on heap
}
```

**Characteristics:**
- Large (limited by available RAM)
- Manual-ish management (garbage collector in JS/Go, manual in C)
- Slower to allocate than stack
- Persistent: data lives until garbage collected or explicitly freed

### Garbage Collection — The Heap Janitor

In JavaScript (and Go), you don't manually free heap memory. The **Garbage Collector (GC)** periodically scans the heap to find objects that no object is referencing anymore, and reclaims that memory.

```
GARBAGE COLLECTION example:

t=0ms: Request A creates `offer` object on heap
t=5ms: Request A sends response, goes out of scope
       `offer` is still on heap — GC hasn't run yet
t=1000ms: GC runs — finds `offer` is unreachable → frees it
t=1000ms: [GC pause: Event Loop stops for 5-50ms]  ← latency spike
```

**The GC pause problem:** When GC runs a full collection, it pauses the entire Node.js Event Loop. Every in-flight request stalls for 5-50ms. This is why keeping heap allocations small and short-lived is important.

---

## 13. System Calls — How Programs Talk to the OS

Your application code (JavaScript, PHP, Go) cannot directly access hardware. It must ask the **Operating System** through **system calls** (syscalls).

### What Syscalls Are

```
APPLICATION (JavaScript/Node.js)
        │
        │  syscall: write()   ← "OS, please send this data on socket 5"
        │  syscall: read()    ← "OS, give me data from socket 5"
        │  syscall: epoll()   ← "OS, tell me when any of these sockets have data"
        ▼
OPERATING SYSTEM KERNEL
        │
        │  controls hardware directly
        ▼
HARDWARE (Network Card, Disk, etc.)
```

### Why This Matters for Non-Blocking I/O

The magic of non-blocking I/O relies on the `epoll` syscall (on Linux):

```
epoll explained:

Normal (blocking):
  thread says: "read from socket" → OS blocks thread until data arrives → thread resumes

epoll (non-blocking):
  thread says: "add socket to watchlist" → returns immediately (non-blocking)
  thread says: "add socket B to watchlist" → returns immediately
  thread says: "add socket C to watchlist" → returns immediately
  thread says: "tell me when ANY socket is ready" → blocks HERE (but on purpose)
  
  Data arrives on socket B → epoll() returns: "socket B is ready"
  thread handles socket B
  thread says "tell me when ANY socket is ready" → waits again
  
  Data arrives on sockets A and C → epoll() returns: "sockets A and C are ready"
  thread handles both

One thread. Thousands of sockets. OS watches all of them simultaneously.
This is libuv's job in Node.js.
```

---

## 14. Network I/O — Talking to Other Machines

In LeadCore, almost every interesting I/O operation is **network I/O** — talking to Redis, MySQL, or Kinesis on other machines inside the same AWS VPC.

### The Journey of a Packet

```
LeadCore sends Redis GET:

Node.js process
    ↓  write() syscall
OS Kernel (TCP/IP stack)
    ↓  builds TCP packet: [IP header | TCP header | "GET budget:123\r\n"]
Network Interface Card (NIC)
    ↓  converts to electrical/optical signals
AWS VPC Internal Network
    ↓  routes to ElastiCache subnet
ElastiCache NIC
    ↓  electrical signals → bytes
Redis process
    ↓  processes command
    ↓  sends response packet

Return journey (same path, reversed)
Total time: 1-3ms  (inside same VPC, same AZ)
```

### Latency Within AWS

```
NETWORK LATENCY REFERENCE (inside AWS us-east-1):

Same AZ, same VPC:                  0.1-1ms    (LeadCore → Redis, MySQL)
Same region, different AZ:          1-3ms
Same region, cross-service:         1-5ms
Inter-region (us-east-1 → us-west): 60-80ms
External internet (→ MAX API):      150-500ms
```

This is why **ElastiCache and Aurora in the same VPC and same AZ as your ECS task** is critical — it minimizes that 1-3ms to 0.1-0.5ms.

### TCP Connection Pooling

Opening a new TCP connection (TCP handshake) takes 1 full round-trip (~1-3ms inside VPC, ~150ms externally). This is why ioredis and mysql2 maintain **connection pools** — reusable open connections so every request doesn't pay the handshake cost.

```
WITHOUT connection pooling:
  Request 1: TCP handshake (1ms) + Redis command (1ms) = 2ms
  Request 2: TCP handshake (1ms) + Redis command (1ms) = 2ms

WITH connection pooling (ioredis does this automatically):
  Startup: TCP handshake once (1ms)
  Request 1: Redis command (1ms) = 1ms  (reuse existing connection)
  Request 2: Redis command (1ms) = 1ms  (reuse existing connection)
```

---

## 15. CPU-Bound vs I/O-Bound Work

This is the classification that determines whether Node.js is the right tool for a given task.

### CPU-Bound Work

The bottleneck is the CPU's speed. More CPU = faster. More I/O speed doesn't help.

```
CPU-BOUND examples for LeadCore:

1. Targeting rule evaluation:
   for each offer (200 offers):
     check state (array lookup — CPU)
     check age range (arithmetic — CPU)
     check tags (set intersection — CPU)
     check dayparting (date math — CPU)
   
   200 offers × 4 checks × ~10 instructions = 8,000 CPU instructions
   At 3 GHz = 2.7 microseconds per offer, 0.5ms for all 200

2. JSON.parse() on large payload:
   Parsing 200 offers × 2KB each = 400KB JSON
   CPU must scan every character, allocate objects
   Takes ~5-15ms of CPU time

3. Sorting 200 qualified offers by bid (comparisons — CPU)

4. Cryptographic click token generation:
   crypto.randomBytes(16) → CPU (hardware random number generator)
   Takes ~0.1ms
```

### I/O-Bound Work

The bottleneck is waiting for external systems. More CPU doesn't help. Better network or faster storage does.

```
I/O-BOUND examples for LeadCore:

1. Redis GET budget:
   CPU work: <1 microsecond (just write bytes to socket)
   Waiting: 1-3ms (network + Redis processing)
   CPU is idle 99.9% of the time for this operation

2. MySQL query for offer data:
   CPU work: <1 microsecond (write SQL to socket)
   Waiting: 5-20ms (network + MySQL query execution)
   CPU is idle 99.9% of the time

3. Kinesis PutRecord:
   CPU work: <1 microsecond
   Waiting: 5-20ms
   CPU is idle 99.9% of the time

4. HTTP response from PHP → LeadCore:
   CPU work: microseconds
   Waiting: round-trip network time
```

### How to Classify Anything

Ask yourself: **"If I gave this operation a faster CPU, would it finish faster?"**

- **Yes → CPU-bound.** Move to Worker Thread if it blocks the event loop.
- **No → I/O-bound.** Async/await handles it perfectly. CPU is idle during wait.

---

## 16. Concurrency vs Parallelism

These two terms are often used interchangeably but they mean very different things.

### Concurrency — Multiple Tasks in Progress

**Concurrency** means multiple tasks are **in progress** at the same time. They may not be executing at the same instant, but they are all started and none is waiting for another to completely finish.

```
CONCURRENCY (one chef, multiple dishes):

Timeline (1 chef):
  t=0:  Start boiling water for Dish A
  t=1:  Start chopping vegetables for Dish B  (water is boiling on its own)
  t=2:  Start marinating meat for Dish C      (water still boiling, veg being chopped)
  t=5:  Water boiled for A → add pasta        (now working on A again)
  t=6:  Back to chopping B
  t=8:  Finish chopping B, start cooking B    (A is cooking on its own)
  ...

Multiple dishes in progress simultaneously.
But at any instant, the chef is doing exactly ONE physical action.
```

### Parallelism — Multiple Tasks Executing Simultaneously

**Parallelism** means multiple tasks are **literally executing at the same instant** on different hardware (different CPU cores).

```
PARALLELISM (multiple chefs, multiple dishes):

Timeline (4 chefs):
  t=0:  Chef 1 boils water for A
        Chef 2 chops vegetables for B  (SIMULTANEOUSLY)
        Chef 3 marinates meat for C    (SIMULTANEOUSLY)
        Chef 4 bakes bread for D       (SIMULTANEOUSLY)

At t=0, all 4 chefs are doing real work at the exact same instant.
4× the throughput of 1 chef (assuming 4 dishes to work on).
```

### The Critical Difference for Node.js

```
NODE.JS Event Loop (CONCURRENT, not parallel for JS):

Request A: starts → Redis call → [suspended at await] → resumes → done
Request B:                        starts → MySQL → [suspended] → resumes → done
Request C:                                          starts → Redis → [suspended] → resumes

All three are "in progress" concurrently.
But at any instant, only one JavaScript function is executing on the Event Loop.
The "waiting" is done by the OS, not the Event Loop.

→ WORKS GREAT for I/O-bound work (CPU is free during the waiting)
→ FAILS for CPU-bound work (CPU must be active for the whole duration)


GO goroutines (PARALLEL):

Request A targeting eval ──► goroutine on Core 1 (running right now)
Request B targeting eval ──► goroutine on Core 2 (running right now, simultaneously)
Request C targeting eval ──► goroutine on Core 3 (running right now, simultaneously)

All three evaluations happen at the literal same instant.
→ BETTER for CPU-bound work
→ Same as Node.js for I/O-bound work
```

### Where the Confusion Comes From

```
Node.js handles 1,000 concurrent connections.
"Concurrent" = 1,000 requests are in-flight (some waiting for Redis, some for MySQL).
The EVENT LOOP handles their callbacks one at a time.
But the OS handles all their I/O simultaneously.

People say "Node.js is concurrent" → true for I/O
People say "Node.js is single-threaded" → true for JavaScript execution
Both are true simultaneously. Not contradictory.
```

---

## 17. How Everything Connects to Node.js and LeadCore

Now let's tie all these concepts together with a complete picture of what happens during a LeadCore feed request.

### A Feed Request, Explained Using Every Term in This Document

```
SCENARIO: PHP sends a preping feed request to LeadCore

Step 1: PHP makes HTTP request to LeadCore
────────────────────────────────────────────────────────────────────
- PHP process (Process, Thread) makes TCP connection to ALB
- ALB routes to LeadCore ECS task (another Process on another machine)
- Network packet travels through VPC (Network I/O)
- LeadCore's NIC receives packet → OS kernel notifies libuv via epoll (Syscall)
- libuv queues callback in Event Loop
- Event Loop (single thread) picks up: new HTTP request arrived

Step 2: Event Loop processes the request
────────────────────────────────────────────────────────────────────
- Fastify parses HTTP body: JSON.parse() on ~2KB body
  → Small enough: 0.01ms CPU work, no blocking concern
  → Memory: request object allocated on Heap

- Check in-memory offer cache:
  → RAM access: 100 nanoseconds (Heap lookup)
  → 200 offer objects already in memory from last cache refresh
  → No I/O needed — instant

Step 3: Concurrent I/O for session, frequency cap, blacklist
────────────────────────────────────────────────────────────────────
- Promise.all([redis.get(session), redis.get(freq), redis.get(blacklist)])
  → 3 non-blocking I/O operations (Syscalls to OS, TCP packets to ElastiCache)
  → Event Loop suspends this request handler (await)
  → Event Loop is FREE → handles other incoming requests during this time
  → OS watches sockets via epoll (1 syscall monitors 3 sockets simultaneously)
  → Packets return from Redis (~2ms) → OS notifies libuv → callbacks queued
  → Event Loop runs callbacks → Promise.all resolves → request handler resumes

Step 4: Targeting evaluation (CPU-Bound)
────────────────────────────────────────────────────────────────────
- 200 offers to evaluate against user profile
- This is CPU work: 8,000 instructions, ~0.5ms
- WITHOUT Worker Threads: runs on Event Loop → blocks for 0.5ms
  → At 100 req/sec: 100 × 0.5ms = 50ms of Event Loop blocked per second
  → Other requests stall during each evaluation
- WITH Worker Threads: sent to Worker Thread (separate OS Thread on another Core)
  → Event Loop is FREE during evaluation
  → Worker uses its own CPU Core (Parallelism for CPU work)
  → Result returned via postMessage (tiny serialization: just user profile)

Step 5: Budget checks (I/O-Bound)
────────────────────────────────────────────────────────────────────
- 50 qualifying offers need budget checks
- Promise.all(50 offers → redis.get(budget:${id}))
  → 50 concurrent non-blocking Redis GETs
  → Event Loop suspends (await Promise.all)
  → libuv sends all 50 packets (ioredis pipelines them = 1-2 TCP packets)
  → OS watches socket via epoll
  → Redis responds with all 50 values (batch response)
  → ~3ms total (not 50 × 3ms = 150ms — concurrent, not sequential)

Step 6: Response building and sending
────────────────────────────────────────────────────────────────────
- Sort 30 affordable offers by bid: tiny CPU work (0.1ms)
- Build response JSON: JSON.stringify() on 30 offers ~60KB
  → ~0.5ms CPU work — on Event Loop (acceptable)
  → Memory: response object on Heap (freed by GC after send)
- HTTP response sent: write() Syscall → OS → NIC → network → PHP

Step 7: Non-critical tracking (fire-and-forget)
────────────────────────────────────────────────────────────────────
- setImmediate(() => kinesis.putRecord(...))
  → Deferred to next Event Loop tick (doesn't delay response)
  → Kinesis write: non-blocking I/O, ~10ms, Event Loop free during it
  → If Kinesis fails: .catch() logs error, doesn't crash process
```

### Timeline of the Entire Request

```
TIME (ms)   EVENT
─────────────────────────────────────────────────────────────────────
0.0ms       Request arrives at Event Loop
0.1ms       HTTP parsed, offer cache read from RAM
0.1ms       3× Redis GETs start (non-blocking, concurrent)
            [Event Loop handles other requests during wait]
2.1ms       Redis responds for all 3 (concurrent → same time)
2.1ms       Targeting work sent to Worker Thread
            [Event Loop handles other requests during eval]
2.6ms       Worker Thread finishes targeting eval (0.5ms CPU)
2.6ms       50× budget Redis GETs start (non-blocking, concurrent)
            [Event Loop handles other requests during wait]
5.6ms       Redis responds for all 50 budget checks
5.7ms       Sort + filter: 0.1ms CPU
6.0ms       JSON.stringify response: 0.3ms CPU
6.0ms       HTTP response sent (write syscall)
6.5ms       setImmediate fires: Kinesis write starts (non-blocking)
─────────────────────────────────────────────────────────────────────
Total P50 latency: ~6ms
Kinesis completes in background at ~16ms (not blocking response)
```

This is LeadCore running correctly: Event Loop free ~95% of the time, only occupied for tiny bursts of actual CPU work.

---

## Quick Reference: All Terms

| Term | One-Line Definition |
|---|---|
| **CPU** | The chip that executes instructions — the brain |
| **CPU Core** | An independent execution unit inside the CPU — one brain can have multiple |
| **vCPU** | AWS's unit — typically one hardware thread of one physical core |
| **Clock Speed (GHz)** | How many instruction cycles per second the CPU can do |
| **RAM** | Fast, temporary memory where running programs live (the desk) |
| **Heap** | The large dynamic region of RAM for objects with unknown size/lifetime |
| **Stack** | The small, ordered region of RAM for function call frames and local vars |
| **Disk/SSD** | Permanent storage, much slower than RAM (the filing cabinet) |
| **Process** | A running instance of a program with its own isolated memory |
| **Thread** | An independent execution stream inside a process; shares memory with others |
| **Context Switch** | OS pausing one thread and resuming another on the same CPU core |
| **I/O** | Any operation that exchanges data with something outside CPU+RAM |
| **Blocking I/O** | Your thread waits and does nothing until the I/O completes |
| **Non-Blocking I/O** | Your thread registers a callback and moves on; OS notifies when done |
| **Syscall** | A request from your app to the OS kernel to access hardware |
| **epoll** | Linux syscall that efficiently monitors thousands of sockets simultaneously |
| **Synchronous** | Operations happen in order, each waits for the previous to finish |
| **Asynchronous** | Operations are started and you come back for the result when ready |
| **Concurrency** | Multiple tasks in progress simultaneously (may interleave on one thread) |
| **Parallelism** | Multiple tasks executing literally simultaneously on different CPU cores |
| **Event Loop** | Node.js's single thread that runs JS code and coordinates I/O callbacks |
| **libuv** | C library under Node.js that handles non-blocking I/O and the event loop |
| **Garbage Collector** | Automatic system that reclaims heap memory no longer reachable |
| **GC Pause** | Event Loop stops during garbage collection — causes latency spikes |
| **CPU-Bound** | Work where the bottleneck is CPU speed (math, logic, parsing) |
| **I/O-Bound** | Work where the bottleneck is waiting for external systems (Redis, MySQL) |
| **Worker Thread** | A separate OS thread in Node.js that runs JS in parallel for CPU work |
| **Promise.all** | Run multiple async operations concurrently, resolve when all finish |
| **Race Condition** | Two threads read+write shared data simultaneously causing wrong results |
| **Connection Pool** | Reusable set of open TCP connections to avoid handshake cost per request |
| **TCP Handshake** | 3-way negotiation to establish a connection (~1ms same VPC) |

---

*Document: Computing Fundamentals for Engineers*
*Version: 1.0*
*Date: April 2026*
*Read this before: `NODEJS_INTERNALS_DEEP_DIVE.md`*
