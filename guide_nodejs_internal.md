# Node.js Internals — How It Really Works

**Document Type:** Engineering Education / Reference
**Author:** LeadCore Team
**Date:** April 2026

---

## TL;DR — What This Document Is About

If you've ever wondered why Node.js is "fast" even though it only uses one thread, or why your Node.js server sometimes freezes under load, or what "non-blocking I/O" actually means in practice — this is the document for you.

> **New to these concepts?** Read `COMPUTING_FUNDAMENTALS_FOR_ENGINEERS.md` first. It explains CPU cores, threads, processes, I/O, RAM, context switching, and concurrency vs parallelism from scratch — with analogies. This document assumes you understand those terms.

We explain how Node.js works under the hood, with real examples from our LeadCore system (Redis lookups, MySQL queries, Kinesis writes, targeting evaluation). By the end, you'll know exactly when Node.js is the right choice, when it will hurt you, and how to fix its limitations.

---

## Table of Contents

1. [The Single Thread — What It Means](#1-the-single-thread--what-it-means)
2. [The Event Loop — The Heart of Node.js](#2-the-event-loop--the-heart-of-nodejs)
3. [Non-Blocking I/O — The Superpower](#3-non-blocking-io--the-superpower)
4. [Async/Await and Promises — How You Write It](#4-asyncawait-and-promises--how-you-write-it)
5. [Concurrency vs Parallelism — The Most Important Distinction](#5-concurrency-vs-parallelism--the-most-important-distinction)
6. [CPU-Bound vs I/O-Bound Work — Why This Changes Everything](#6-cpu-bound-vs-io-bound-work--why-this-changes-everything)
7. [The Event Loop Blocking Problem — When Node.js Fails](#7-the-event-loop-blocking-problem--when-nodejs-fails)
8. [Worker Threads — The Fix for CPU-Bound Work](#8-worker-threads--the-fix-for-cpu-bound-work)
9. [Real LeadCore Examples — Putting It All Together](#9-real-leadcore-examples--putting-it-all-together)
10. [CPU Utilization Deep Dive](#10-cpu-utilization-deep-dive)
11. [Limitations Summary and How to Overcome Them](#11-limitations-summary-and-how-to-overcome-them)
12. [Mental Models — How to Think About Node.js](#12-mental-models--how-to-think-about-nodejs)

---

## 1. The Single Thread — What It Means

### What a Thread Is

A thread is an independent sequence of instructions that a CPU core can execute. Your operating system can run multiple threads simultaneously, one per CPU core.

Most web frameworks (PHP-FPM, Java Spring, Python Gunicorn) handle concurrency like this:

```
TRADITIONAL MULTI-THREADED SERVER (e.g., PHP-FPM)

Incoming requests → Thread pool

Request 1 → Thread 1 (dedicated) → handles everything → done → thread released
Request 2 → Thread 2 (dedicated) → handles everything → done → thread released
Request 3 → Thread 3 (dedicated) → handles everything → done → thread released
...

If 100 requests come in simultaneously → 100 threads needed
If thread pool is 50 threads → requests 51-100 wait in queue
```

Each thread consumes:
- ~1-8MB of memory (stack space)
- OS scheduling overhead (context switching)
- CPU time even while idle (waiting for database)

**The fundamental inefficiency:** If Thread 1 is waiting for a database query (taking 5ms), it's doing nothing — but still consuming a thread slot and blocking other requests from using it.

### Node.js Takes a Different Approach

Node.js runs your JavaScript code on **a single thread** — but it delegates all waiting (I/O) to the OS and libuv library. The single thread never waits. It just keeps working.

```
NODE.JS SINGLE-THREADED SERVER

Incoming requests → Single Event Loop Thread

Request 1 arrives → start processing → needs Redis → "go get it, call me when done" → move on
Request 2 arrives → start processing → needs MySQL → "go get it, call me when done" → move on
Request 3 arrives → start processing → needs Kinesis → "go write it, call me when done" → move on

...later...

Redis responds for Request 1 → Event Loop picks it up → finish Request 1 → send response
MySQL responds for Request 2 → Event Loop picks it up → finish Request 2 → send response
```

The single thread is never idle. It processes in tiny bursts, context-switching between requests at each I/O boundary.

**Memory cost:** ~80MB for the whole process (not per request).
**Thread cost:** 1 OS thread for JavaScript, a few background threads in libuv for system calls.

---

## 2. The Event Loop — The Heart of Node.js

The event loop is the mechanism that makes this possible. It is a continuous loop that asks: "Is there anything ready to run?" If yes, it runs it. If no, it waits.

### How the Event Loop Works (Simplified)

```
EVENT LOOP — One Iteration ("Tick")

┌─────────────────────────────────────────────────────┐
│  Phase 1: Timers                                    │
│  Run callbacks scheduled by setTimeout/setInterval  │
│  that have reached their deadline                   │
├─────────────────────────────────────────────────────┤
│  Phase 2: Pending Callbacks                         │
│  Run I/O callbacks deferred from previous tick      │
├─────────────────────────────────────────────────────┤
│  Phase 3: Poll                                      │
│  Wait for new I/O events and execute their callbacks│
│  (This is where Node spends most of its time)       │
├─────────────────────────────────────────────────────┤
│  Phase 4: Check                                     │
│  Run setImmediate() callbacks                       │
├─────────────────────────────────────────────────────┤
│  Phase 5: Close Callbacks                           │
│  Run close event callbacks (socket.on('close'))     │
└─────────────────────────────────────────────────────┘
               ↓ repeat forever
```

Between each phase, Node.js also checks `process.nextTick()` and resolved Promises (microtasks) and runs those first.

### A Concrete Timeline

Let's trace what happens when 3 requests arrive within milliseconds of each other:

```
TIME    EVENT
──────────────────────────────────────────────────────────────────────
0ms     Request A arrives → Event Loop starts processing A
0.1ms   Request A needs Redis GET → sends async request to Redis
        Event Loop is free now — A is "parked" waiting for Redis

0.1ms   Request B arrives → Event Loop starts processing B
0.2ms   Request B needs MySQL query → sends async query to MySQL
        Event Loop is free now — B is "parked" waiting for MySQL

0.2ms   Request C arrives → Event Loop starts processing C
0.3ms   Request C needs Redis GET → sends async request to Redis
        Event Loop is free now — C is "parked" waiting for Redis

        [Event Loop polls: "anything ready?" — No → poll, wait]

2.1ms   Redis responds for Request A → Event Loop picks up callback
        Processes A's response → sends HTTP response to client
        Request A complete ✓

3.5ms   Redis responds for Request C → Event Loop picks up callback
        Processes C's response → sends HTTP response to client
        Request C complete ✓

5.2ms   MySQL responds for Request B → Event Loop picks up callback
        Processes B's response → sends HTTP response to client
        Request B complete ✓
```

All 3 requests were handled by the same single thread, overlapping in time. Total time: 5.2ms. If they were sequential (blocking), it would be 2+3.5+5 = 10.5ms.

---

## 3. Non-Blocking I/O — The Superpower

### What "Blocking" Means

Blocking means: your thread stops and waits until the operation finishes.

```php
// PHP — BLOCKING (traditional)
$result = $redis->get('budget:campaign:123');  // Thread STOPS HERE
// Thread is frozen until Redis responds (2ms)
// No other request can use this thread during those 2ms
echo $result;
```

```javascript
// Node.js (BAD — synchronous Redis, hypothetical)
const result = redis.getSync('budget:campaign:123'); // Would BLOCK if this existed
// Event Loop is frozen. ALL other requests wait.
```

### What "Non-Blocking" Means

Non-blocking means: register a callback (or await a Promise), tell the OS to do the work, and immediately return. The event loop continues processing other things. When the OS finishes the I/O, it notifies Node.js, which runs the callback.

```javascript
// Node.js — NON-BLOCKING (real)
const result = await redis.get('budget:campaign:123');
// "await" suspends THIS function's execution
// BUT the Event Loop is free to handle other requests
// When Redis responds, this function resumes from here
console.log(result); // runs after Redis responds
```

### Under the Hood: How Non-Blocking I/O Actually Works

When you do `redis.get(...)`, here is what actually happens:

```
Your Code             libuv (C library)         OS Kernel         Redis Server
────────────────────────────────────────────────────────────────────────────
redis.get("key")
    │
    ├──► libuv: make TCP write  ──────────────────────────────► [GET key\r\n]
    │
    │    [your JS callback is stored in libuv's event table]
    │    [execution returns to Event Loop immediately]
    │
    │    [Event Loop processes other requests...]
    │
    │                          OS: data arrived on socket ◄──── [*bulk 5\r\nvalue\r\n]
    │                              ↓
    │                          libuv: "callback registered for this socket"
    │                              ↓
    │                          libuv pushes callback to Event Loop queue
    │
    ◄── Event Loop calls your callback with result
    │
    console.log(result)  // "value"
```

The key insight: **libuv uses OS-level mechanisms** (`epoll` on Linux, `kqueue` on macOS) that can monitor thousands of sockets simultaneously without threads. The OS tells libuv "this socket has data" via a single system call, libuv queues the callback, and Node.js runs it.

This is how Node.js can handle thousands of concurrent connections with one thread.

---

## 4. Async/Await and Promises — How You Write It

### Promises: The Foundation

A Promise represents a value that will be available in the future.

```javascript
// A function that returns a Promise
function getRedisValue(key) {
    return new Promise((resolve, reject) => {
        // This is where the non-blocking I/O happens
        redisClient.get(key, (error, result) => {
            if (error) reject(error);
            else resolve(result);
        });
    });
}

// Using it with .then() chaining
getRedisValue('budget:123')
    .then(value => {
        console.log('Got:', value);
        return processValue(value);  // can return another Promise
    })
    .then(processed => {
        console.log('Processed:', processed);
    })
    .catch(error => {
        console.error('Error:', error);
    });
```

### Async/Await: Syntactic Sugar Over Promises

`async/await` is just a cleaner way to write the same thing. Under the hood, it compiles to Promises:

```javascript
// Same logic as above, but readable like synchronous code
async function handleBudgetCheck(campaignId) {
    try {
        const value = await getRedisValue(`budget:${campaignId}`);
        // Code PAUSES here (non-blocking — Event Loop is free)
        // Resumes when Redis responds
        
        const processed = await processValue(value);
        // Code PAUSES here again
        // Resumes when processValue resolves
        
        return processed;
    } catch (error) {
        console.error('Error:', error);
    }
}
```

**Critical mental model:** Every `await` is a "suspension point." The current function pauses, but the Event Loop continues running other callbacks and handling other requests.

### Sequential vs Concurrent Async

This is where most developers make costly mistakes:

```javascript
// SEQUENTIAL — each awaits the previous one (SLOW)
// Total time = time(Redis) + time(MySQL) + time(Kinesis)
async function sequentialExample(userId) {
    const budget = await redis.get(`budget:${userId}`);       // wait 2ms
    const campaign = await mysql.query('SELECT...', [userId]);// wait 5ms
    const rules = await redis.get(`rules:${userId}`);         // wait 2ms
    // Total: 9ms
    return { budget, campaign, rules };
}

// CONCURRENT — all start at the same time (FAST)
// Total time = max(time(Redis), time(MySQL), time(Kinesis))
async function concurrentExample(userId) {
    const [budget, campaign, rules] = await Promise.all([
        redis.get(`budget:${userId}`),           // starts immediately
        mysql.query('SELECT...', [userId]),       // starts immediately
        redis.get(`rules:${userId}`)             // starts immediately
    ]);
    // Total: 5ms (the slowest one)
    return { budget, campaign, rules };
}
```

Real-world impact in our LeadCore preping flow:

```
SEQUENTIAL (wrong approach):
  Redis check:  2ms
  MySQL load:   8ms
  Rules load:   3ms
  Total:       13ms

CONCURRENT with Promise.all (right approach):
  All three start at t=0ms, finish at t=8ms (MySQL is the bottleneck)
  Total:        8ms  ← 38% faster with zero extra infrastructure
```

---

## 5. Concurrency vs Parallelism — The Most Important Distinction

This is the single most important concept for understanding Node.js's strengths and weaknesses.

### Definitions

**Concurrency:** Multiple tasks are in progress at the same time, but not necessarily executing simultaneously. They interleave on shared resources.

**Parallelism:** Multiple tasks are literally executing at the exact same instant on different CPU cores.

### The Restaurant Analogy

```
PARALLELISM (Go, Java with threads):
  10 chefs in kitchen
  Chef 1 cooks Dish A
  Chef 2 cooks Dish B simultaneously
  Chef 3 cooks Dish C simultaneously
  All cooking happens at the exact same moment

CONCURRENCY (Node.js Event Loop):
  1 chef in kitchen
  Chef starts Dish A, puts it in oven (oven = OS/network I/O)
  Chef starts Dish B, puts it in oven
  Chef starts Dish C, puts it in oven
  [Chef checks: Dish A done? Dish B done? Dish C done?]
  Dish B oven beeps → Chef finishes Dish B
  Dish A oven beeps → Chef finishes Dish A
  Dish C oven beeps → Chef finishes Dish C
  All dishes served. Chef was always doing something useful.
```

The chef is never idle — but the chef can only do one physical action at a time. The "oven" (OS kernel) does the waiting.

### Why This Matters for Node.js

```
WAITING FOR REDIS (I/O — oven):
  Node.js sends request, OS waits for network response
  Event Loop moves on, handles other requests
  When Redis responds, callback runs
  → Node.js is GREAT at this — concurrency is enough

EVALUATING TARGETING RULES (CPU — chef cooking):
  Node.js must do the calculation itself — no "oven" to delegate to
  Event Loop is OCCUPIED during this entire time
  Other requests WAIT
  → Node.js is LIMITED here — it needs real parallelism (Worker Threads)
```

### Timing Comparison for Our Feed Request (200 offers):

```
SCENARIO: Evaluate 200 offers for a user, each taking 0.02ms of CPU

WITH GO (true parallelism, 2 cores):
  Core 1: evaluates offers 1-100 in parallel → done in ~1ms
  Core 2: evaluates offers 101-200 in parallel → done in ~1ms
  Total: ~1ms (both cores working simultaneously)

WITH NODE.JS (concurrency, 1 thread):
  Event Loop: evaluates offer 1 → offer 2 → offer 3 → ... → offer 200
  Total: 200 × 0.02ms = 4ms (sequential — one at a time)

WITH NODE.JS + WORKER THREADS (2 workers):
  Worker 1: evaluates offers 1-100 → done in ~2ms
  Worker 2: evaluates offers 101-200 → done in ~2ms
  Total: ~2ms (true parallelism restored)
```

For 200 offers and your current traffic, 4ms is fine. At 2,000 req/sec with 200 offers each, it becomes a bottleneck.

---

## 6. CPU-Bound vs I/O-Bound Work — Why This Changes Everything

Every operation your code does falls into one of two categories:

### I/O-Bound: Waiting for Something External

Examples:
- Reading from Redis
- Querying MySQL/Aurora
- Writing to Kinesis
- Making an HTTP request to another service
- Reading from disk
- Accepting a network connection

**Characteristic:** Your CPU is idle during this work. The OS, network card, or disk is doing the actual work. The thread (or event loop) just waits for the result.

**Node.js behavior:** Perfect. Sends the request, registers a callback, moves on. The Event Loop handles thousands of these concurrently. CPU usage stays low.

```javascript
// I/O-Bound work — Node.js is excellent
async function fetchUserData(userId) {
    // Node.js is doing NOTHING while Redis processes this
    // Event Loop is free to handle 500 other requests simultaneously
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);
    
    // Still I/O-bound — Event Loop free while MySQL runs query
    const user = await mysql.query('SELECT * FROM users WHERE id = ?', [userId]);
    
    // Write back to cache — I/O-bound
    await redis.setex(`user:${userId}`, 300, JSON.stringify(user));
    
    return user;
}
```

### CPU-Bound: Your Thread Does the Work

Examples:
- Evaluating targeting rules (logic, comparisons, loops)
- JSON.parse() on a large payload
- Cryptographic operations (hashing, signing)
- Image resizing
- Regular expressions on long strings
- Sorting large arrays
- Mathematical calculations

**Characteristic:** Your CPU is doing actual work. No waiting. The thread is 100% occupied.

**Node.js behavior:** The Event Loop is blocked for the duration. All other requests wait. No other JavaScript can execute until this finishes.

```javascript
// CPU-Bound work — Node.js struggles
function evaluateAllOffers(offers, userProfile) {
    // This loop runs entirely on the Event Loop thread
    // For 200 offers × 0.02ms each = 4ms of Event Loop blockage
    // During these 4ms, no other request is processed
    return offers.filter(offer => {
        const locationMatch = offer.states.includes(userProfile.state);
        const ageMatch = userProfile.age >= offer.minAge && userProfile.age <= offer.maxAge;
        const tagMatch = offer.tags.some(tag => userProfile.tags.includes(tag));
        const daypartMatch = checkDayparting(offer.schedule, userProfile.timezone);
        return locationMatch && ageMatch && tagMatch && daypartMatch;
    });
}
```

### Measuring the Difference

```javascript
// A simple benchmark to feel the difference

const { performance } = require('perf_hooks');

// I/O-Bound simulation (correct — non-blocking)
async function ioBoundWork() {
    const start = performance.now();
    
    // Simulate 100 concurrent Redis lookups
    const promises = Array(100).fill(0).map((_, i) =>
        redis.get(`key:${i}`)  // All start simultaneously, none block
    );
    await Promise.all(promises);
    
    console.log(`I/O-bound: ${(performance.now() - start).toFixed(2)}ms`);
    // Output: I/O-bound: 3.2ms  (all 100 ran concurrently, Redis RTT ~3ms)
}

// CPU-Bound simulation (blocks event loop)
function cpuBoundWork() {
    const start = performance.now();
    
    // Simulate evaluating 10,000 offers
    const offers = Array(10000).fill({ age: [18, 65], states: ['NY', 'CA', 'TX'] });
    const user = { age: 35, state: 'NY', tags: ['medicare'] };
    
    const results = offers.filter(o =>
        o.age[0] <= user.age &&
        user.age <= o.age[1] &&
        o.states.includes(user.state)
    );
    
    console.log(`CPU-bound: ${(performance.now() - start).toFixed(2)}ms`);
    // Output: CPU-bound: 8.7ms  (Event Loop was BLOCKED for 8.7ms)
    return results;
}
```

---

## 7. The Event Loop Blocking Problem — When Node.js Fails

### What Happens When the Event Loop Blocks

```
NORMAL (Event Loop free):

Time ──────────────────────────────────────────────────────────────►
      Req A   Req A   Req B   Req A   Req B   Req C   Req C
      start   wait    start   done    wait    start   done
              [Redis] [Redis] ✓       [MySQL] [Redis]  ✓
                      ▼               ▼
                   Req B done ✓    Req A (MySQL) done ✓

Each request gets tiny slices, responds quickly.


BLOCKED Event Loop (CPU work happening):

Time ──────────────────────────────────────────────────────────────►
      ████████████████████████████████████   Req A   Req B   Req C
      CPU WORK FOR REQUEST X (8ms)           done    done    done
      [Event Loop cannot process ANYTHING]

8ms of complete stall. All incoming requests sit in queue.
```

### Real Example: The JSON Parsing Trap

In LeadCore, we refresh the offer cache every 60 seconds by fetching 200+ offers from MySQL and storing them in memory. If we do this on the main thread:

```javascript
// DANGEROUS — runs on Event Loop thread
async function refreshOfferCache() {
    const rawOffers = await mysql.query('SELECT * FROM lc_offers WHERE active = 1');
    // ↑ This is I/O-bound — fine, Event Loop is free during query

    // ↓ THIS IS THE PROBLEM — JSON parse of 200 complex offer objects
    const parsed = rawOffers.map(row => ({
        ...row,
        targeting: JSON.parse(row.targeting_json),  // CPU work
        schedule: JSON.parse(row.schedule_json),     // CPU work
        bids: JSON.parse(row.bids_json)              // CPU work
    }));
    // If this takes 15ms, the Event Loop is blocked for 15ms

    offerCache = parsed;
}
```

Every 60 seconds, all requests stall for 15ms. At 100 req/sec, 1-2 requests are delayed by 15ms. Invisible. But at 1,000 req/sec, ~15 requests queue up during that 15ms window — your P99 latency spikes every 60 seconds.

### Real Example: Regex on Large Strings

```javascript
// Someone adds a "smart" user agent parser
function detectDevice(userAgent) {
    // These regexes run on the Event Loop
    const isMobile = /Mobile|Android|iPhone|iPad|iPod|BlackBerry/i.test(userAgent);
    const isBot = /Googlebot|Bingbot|Slurp|DuckDuckBot|facebookexternalhit/i.test(userAgent);
    
    // For short strings: fast (<0.1ms) — fine
    // If userAgent is 5000 chars with a catastrophic backtracking regex: 50ms+ — disaster
    return { isMobile, isBot };
}
```

### Measuring Event Loop Lag

```javascript
// This tells you how much the Event Loop is being delayed
let lastCheck = Date.now();
setInterval(() => {
    const now = Date.now();
    const lag = now - lastCheck - 100; // Should be ~0ms, we set 100ms interval
    if (lag > 10) {
        console.warn(`Event Loop lag: ${lag}ms — something is blocking!`);
    }
    lastCheck = now;
}, 100);
```

Put this in your LeadCore service. If lag exceeds 10ms, you know the Event Loop is being blocked somewhere.

---

## 8. Worker Threads — The Fix for CPU-Bound Work

`worker_threads` (built into Node.js since v10.5) let you run JavaScript on separate OS threads — true parallelism for CPU-bound work.

### How Worker Threads Differ from the Event Loop

```
WITHOUT WORKER THREADS:
  Process
  └── Main Thread (Event Loop)
      ├── handles HTTP requests
      ├── handles Redis callbacks
      ├── handles MySQL callbacks
      └── does CPU work (blocks everything above while running)


WITH WORKER THREADS:
  Process
  ├── Main Thread (Event Loop)
  │   ├── handles HTTP requests
  │   ├── handles Redis callbacks
  │   ├── handles MySQL callbacks
  │   └── sends CPU work to Worker → never blocks
  │
  ├── Worker Thread 1 (separate OS thread)
  │   └── does CPU work (targeting evaluation)
  │
  └── Worker Thread 2 (separate OS thread)
      └── does CPU work (targeting evaluation)
```

### A Worker Thread Pool for Targeting Evaluation

```javascript
// targeting.worker.js — runs in a separate thread
const { workerData, parentPort } = require('worker_threads');

function evaluateTargeting(offers, userProfile) {
    return offers.filter(offer => {
        const locationOk = offer.states.includes(userProfile.state);
        const ageOk = userProfile.age >= offer.minAge && userProfile.age <= offer.maxAge;
        const tagOk = offer.tags.some(t => userProfile.tags.has(t));
        return locationOk && ageOk && tagOk;
    });
}

// Worker receives tasks via message
parentPort.on('message', ({ taskId, offers, userProfile }) => {
    const result = evaluateTargeting(offers, userProfile);
    parentPort.postMessage({ taskId, result });
});
```

```javascript
// worker-pool.js — in main thread
const { Worker } = require('worker_threads');

class TargetingWorkerPool {
    constructor(numWorkers) {
        this.workers = [];
        this.queue = [];
        this.taskCallbacks = new Map();
        this.taskCounter = 0;

        for (let i = 0; i < numWorkers; i++) {
            const worker = new Worker('./targeting.worker.js');
            worker.on('message', ({ taskId, result }) => {
                const { resolve } = this.taskCallbacks.get(taskId);
                this.taskCallbacks.delete(taskId);
                resolve(result);
                this.processQueue(worker);
            });
            this.workers.push({ worker, busy: false });
        }
    }

    evaluateTargeting(offers, userProfile) {
        return new Promise((resolve, reject) => {
            const taskId = this.taskCounter++;
            this.taskCallbacks.set(taskId, { resolve, reject });
            
            const freeWorker = this.workers.find(w => !w.busy);
            if (freeWorker) {
                freeWorker.busy = true;
                freeWorker.worker.postMessage({ taskId, offers, userProfile });
            } else {
                this.queue.push({ taskId, offers, userProfile });
            }
        });
    }

    processQueue(workerState) {
        if (this.queue.length === 0) {
            workerState.busy = false;
            return;
        }
        const { taskId, offers, userProfile } = this.queue.shift();
        workerState.worker.postMessage({ taskId, offers, userProfile });
    }
}

// Usage in main thread (Event Loop is NOT blocked)
const pool = new TargetingWorkerPool(4); // 4 worker threads

app.get('/feed', async (req, res) => {
    const offers = getOffersFromCache(); // instant — in-memory
    
    // This sends work to a Worker thread and awaits the result
    // Event Loop is FREE during evaluation — other requests processed normally
    const qualified = await pool.evaluateTargeting(offers, req.userProfile);
    
    const budgets = await Promise.all(
        qualified.map(o => redis.get(`budget:${o.id}`)) // I/O-bound — concurrent
    );
    
    res.json(buildFeedResponse(qualified, budgets));
});
```

### The Serialization Cost (Important Limitation of Worker Threads)

Worker threads cannot share memory directly (unless you use `SharedArrayBuffer`). Data passed via `postMessage` is **copied** (serialized to binary, transferred, deserialized). This has a cost:

```
Data being sent to Worker Thread:
  200 offers × ~2KB each = ~400KB of offer data
  User profile = ~1KB

Serialization cost: ~0.5ms (one-way)
Deserialization cost in worker: ~0.5ms
Return result: 50 qualifying offers × ~2KB = 100KB → ~0.1ms

Total overhead just from copying data: ~1.1ms

This overhead is fixed per request. If targeting eval itself is 2ms,
Worker Thread total = 1.1ms + 2ms + 1.1ms = 4.2ms (vs 2ms if it were free)
Worth it only if the CPU work takes > 2ms AND it was blocking the Event Loop.
```

**Optimization:** Keep a copy of the offer cache in each Worker Thread. Only send the user profile (1KB), not the 200 offers (400KB). Workers already have the data they need.

```javascript
// Efficient: each worker holds the offer cache
// Only send user profile across the thread boundary

// In worker: receive cache updates
parentPort.on('message', ({ type, data }) => {
    if (type === 'UPDATE_CACHE') {
        offerCache = data;  // ~400KB, but only every 60 seconds
    } else if (type === 'EVALUATE') {
        const { taskId, userProfile } = data;
        const result = evaluateTargeting(offerCache, userProfile); // use local cache
        parentPort.postMessage({ taskId, result });
        // Now only ~1KB crosses the thread boundary per request
    }
});
```

---

## 9. Real LeadCore Examples — Putting It All Together

### Example 1: The Preping Feed Call (Our Heaviest Request)

This is what happens when our PHP app calls LeadCore's `/feed?ispreping=1`:

```javascript
// LeadCore /feed handler
app.post('/feed', async (req, res) => {
    const { userId, sessionId, userProfile, numOfAds } = req.body;

    // ─── Phase 1: Load data (ALL I/O-Bound — run concurrently) ───────────────
    const [
        cachedSession,    // Redis GET — 1-3ms
        frequencyCap,     // Redis GET — 1-3ms
        userBlacklist,    // Redis GET — 1-3ms
    ] = await Promise.all([
        redis.get(`session:${sessionId}`),
        redis.get(`freq:${userId}`),
        redis.get(`blacklist:${userId}`)
    ]);
    // Total: ~3ms (slowest of three, they ran in parallel)

    if (frequencyCap > MAX_FREQUENCY) {
        return res.json({ ads: [] });
    }

    // ─── Phase 2: Targeting Evaluation (CPU-Bound) ────────────────────────────
    const activeOffers = offerCache.get();  // in-memory, instant ~0.1ms

    // If no worker threads: this takes ~4ms and BLOCKS the Event Loop
    // const qualified = activeOffers.filter(o => evaluateTargeting(o, userProfile));

    // With worker threads: takes ~2ms, Event Loop FREE during evaluation
    const qualified = await workerPool.evaluateTargeting(activeOffers, userProfile);

    // ─── Phase 3: Budget Check (I/O-Bound — run concurrently) ────────────────
    const budgetResults = await Promise.all(
        qualified.map(offer =>
            redis.get(`budget:spent:${offer.campaignId}`)
        )
    );
    // Total: ~3ms (all budget checks run in parallel — one Redis RTT)

    const affordable = qualified.filter((offer, i) => {
        const spent = parseFloat(budgetResults[i] || '0');
        return spent < offer.dailyBudget;
    });

    // ─── Phase 4: Build response ──────────────────────────────────────────────
    const topN = affordable
        .sort((a, b) => b.bid - a.bid)
        .slice(0, numOfAds);

    // Fire impression tracking async — don't wait for it (non-critical path)
    setImmediate(() => {
        kinesis.putRecord({
            StreamName: 'leadcore-events',
            Data: JSON.stringify({ type: 'preping_impression', userId, sessionId, offers: topN.map(o => o.id) }),
            PartitionKey: sessionId
        }).promise().catch(err => logger.error('Kinesis error', err));
    });

    // ─── Phase 5: Respond ─────────────────────────────────────────────────────
    return res.json({
        ads: topN.map(o => ({
            offerId: o.id,
            bid: o.bid,
            advertiser: o.advertiserId
        }))
    });
});

// Total latency breakdown:
// Phase 1 (parallel Redis):      3ms
// Phase 2 (targeting workers):   2ms
// Phase 3 (parallel budget):     3ms
// Phase 4 (sort/slice):          0.1ms
// HTTP overhead:                 1ms
// ─────────────────────────────────
// Total P50:                    ~9ms
// Total P99:                   ~20ms (Redis + MySQL variance)
```

### Example 2: Budget Decrement (Atomic, Non-Blocking)

```javascript
// Budget decrement via Redis Lua — atomic + non-blocking
const budgetDecrementScript = `
    local spent_key = KEYS[1]
    local cap_key   = KEYS[2]
    local amount    = tonumber(ARGV[1])
    
    local current = tonumber(redis.call('GET', spent_key) or '0')
    local cap     = tonumber(redis.call('GET', cap_key) or '0')
    
    if cap > 0 and current + amount > cap then
        return {0, current, cap}  -- rejected: over budget
    end
    
    local new_spent = redis.call('INCRBYFLOAT', spent_key, amount)
    return {1, new_spent, cap}  -- approved
`;

async function decrementBudget(campaignId, bidAmount) {
    // This sends the Lua script to Redis — runs atomically on Redis server
    // Non-blocking: Event Loop is FREE while Redis processes it
    const [approved, newSpent, cap] = await redis.eval(
        budgetDecrementScript,
        2,
        `budget:spent:${campaignId}`,
        `budget:cap:${campaignId}`,
        bidAmount.toString()
    );
    
    return {
        approved: approved === 1,
        newSpent: parseFloat(newSpent),
        cap: parseFloat(cap)
    };
}

// How it performs:
// - Atomic: no race conditions (Lua runs as single Redis command)
// - Non-blocking: Event Loop free during Redis processing (~1-2ms)
// - No Node.js CPU used: all logic runs on Redis server
// This is Node.js + Redis at their best
```

### Example 3: Click Tracking (Fire and Forget)

```javascript
// /lc/click/:token — must be FAST, user is waiting for redirect
app.get('/lc/click/:token', async (req, res) => {
    const token = req.params.token;

    // Validate token — Redis GET (non-blocking, ~2ms)
    const clickData = await redis.get(`click:${token}`);
    
    if (!clickData) {
        return res.status(404).send('Invalid click token');
    }

    const { advertiserId, campaignId, destinationUrl, userId } = JSON.parse(clickData);

    // Generate conversion tracking ID (CPU — but trivially fast ~0.01ms)
    const conversionId = crypto.randomBytes(16).toString('hex');
    
    // Store for conversion tracking — Redis SET (non-blocking)
    // Use Promise, NOT await — we don't need to wait for this to redirect
    redis.setex(`conv:pending:${conversionId}`, 86400, JSON.stringify({
        advertiserId, campaignId, userId, clickedAt: Date.now()
    })).catch(err => logger.error('Redis error on click tracking', err));

    // Emit event to Kinesis — also fire and forget
    kinesis.putRecord({
        StreamName: 'leadcore-events',
        Data: JSON.stringify({ type: 'click', token, conversionId, campaignId, userId }),
        PartitionKey: campaignId
    }).promise().catch(err => logger.error('Kinesis click event error', err));

    // Set x-cid header (critical for advertiser postback matching)
    res.setHeader('x-cid', `lc_${conversionId}`);
    
    // Redirect immediately — total latency ~2-3ms (just the Redis GET)
    // The fire-and-forget tasks complete in background without blocking response
    return res.redirect(302, destinationUrl);
});
```

The key pattern here: we `await` only what we need to respond, and fire-and-forget the rest. The user gets redirected in 2-3ms. The Kinesis and Redis writes complete in background.

---

## 10. CPU Utilization Deep Dive

### How CPU Usage Looks for Node.js

```
CPU USAGE GRAPH — I/O-Bound Node.js Server (Healthy)

100% ┤
 90% ┤
 80% ┤
 70% ┤
 60% ┤
 50% ┤                    ▄
 40% ┤                ▄ ▄ █ ▄
 30% ┤            ▄ ▄ █ █ █ █ ▄ ▄
 20% ┤    ▄ ▄ ▄ ▄ █ █ █ █ █ █ █ █ ▄ ▄
 10% ┤ ▄ ▄ █ █ █ █ █ █ █ █ █ █ █ █ █ ▄
  0% ┼─────────────────────────────────────
     Low traffic           High traffic

Node.js CPU scales smoothly with load.
Never hits 100% on one core for I/O-bound work.
```

```
CPU USAGE GRAPH — CPU-Bound Node.js (Problematic)

100% ┤          ██████          ██████
 90% ┤          ██████          ██████
 80% ┤          ██████          ██████
 70% ┤    ████  ██████  ████    ██████  ████
 60% ┤    ████  ██████  ████    ██████  ████
 50% ┤ ██ ████  ██████  ████ ██ ██████  ████
 40% ┤ ██ ████  ██████  ████ ██ ██████  ████
 30% ┤ ██ ████  ██████  ████ ██ ██████  ████
  0% ┼─────────────────────────────────────
     CPU spikes at 100% during CPU-bound operations
     Event Loop blocked during spikes
     Latency spikes correlate with CPU spikes
```

### How Node.js Uses Multiple Cores (Or Doesn't)

A single Node.js process only uses **one CPU core** for JavaScript. If you have a 4-core machine and run one Node.js process, 3 cores sit idle.

Options to use multiple cores:

```javascript
// Option 1: Node.js Cluster module (built-in)
// Creates multiple processes (not threads), each with its own Event Loop
// Each process handles a share of incoming connections

const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
    const numCPUs = os.cpus().length; // e.g., 4
    console.log(`Forking ${numCPUs} workers`);
    
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork(); // Creates child process with full Node.js runtime
    }
    
    cluster.on('exit', (worker, code) => {
        console.log(`Worker ${worker.id} died, restarting...`);
        cluster.fork();
    });
} else {
    // This code runs in each child process
    app.listen(8080);
}

// Result: 4 independent Node.js processes, each handling ~25% of requests
// Each uses one CPU core
// Good: simple, uses all cores
// Bad: 4× memory usage (each process has its own V8 heap + offer cache)
```

```javascript
// Option 2: Worker Threads (better for LeadCore)
// One main process (Event Loop) + worker threads for CPU work
// Workers share address space (memory efficient, but need careful sync)

const { Worker } = require('worker_threads');

// 3 workers on a 4-core machine:
// Core 0: Main Event Loop (HTTP, Redis, MySQL I/O)
// Core 1: Worker 1 (targeting evaluation)
// Core 2: Worker 2 (targeting evaluation)
// Core 3: Worker 3 (targeting evaluation)
```

### Memory and CPU Per Request

```
MEMORY PER REQUEST IN NODE.JS:

Fixed (process lifetime):
  V8 heap (offer cache, routing tables):  ~50MB
  node_modules:                           ~30MB
  Worker thread overhead:                 ~10MB × workers

Per-request (garbage collected after response):
  Request object:                         ~5KB
  Response object:                        ~3KB
  Promise chain allocations:              ~2KB
  Total per request:                      ~10KB

At 100 concurrent requests: 100 × 10KB = 1MB extra heap
Garbage collector reclaims this after requests complete

COMPARE TO PHP-FPM:
  Each PHP process: ~30-60MB (entire runtime per worker)
  At 100 concurrent: 100 × 30MB = 3GB
  Node.js is dramatically more memory-efficient for concurrency
```

### Garbage Collection Impact

Node.js uses V8's garbage collector, which runs periodically to reclaim memory. **GC pauses the Event Loop.**

```
GC events in a typical Node.js server:

Minor GC (young generation): every few seconds, ~1-5ms pause
Major GC (full heap):        every few minutes, ~10-50ms pause

During a 50ms full GC, ALL requests stall.
```

Mitigation:
- Keep heap size small (don't cache huge objects in memory)
- Use `--max-old-space-size` to tune heap limits
- Monitor GC frequency with `--expose-gc` and `gc-stats` npm package
- Use `Buffer` for binary data (outside V8 heap, not GC'd)

---

## 11. Limitations Summary and How to Overcome Them

### Limitation 1: Single-Threaded CPU

**Problem:** CPU-bound work blocks all requests.
**When it hurts:** Targeting rule evaluation at high concurrency, JSON parsing of large payloads, cryptographic work.
**Fix:** Worker Thread pool. CPU work runs on separate OS threads.

```javascript
// Before (bad): blocks Event Loop for 4ms per request
const results = offers.filter(o => evaluateTargeting(o, profile));

// After (good): Event Loop free, Worker does CPU work in parallel
const results = await workerPool.evaluateTargeting(offers, profile);
```

---

### Limitation 2: No Built-in Request Isolation

**Problem:** In PHP-FPM, each request has its own process. A crash kills only that request. In Node.js, an unhandled exception can crash the entire process — all in-flight requests fail.

**Fix:** Global error handlers + process manager + health checks.

```javascript
// Catch uncaught exceptions — NEVER let these kill the process silently
process.on('uncaughtException', (error) => {
    logger.error('Uncaught exception — this should never happen!', { error });
    // Optionally: graceful shutdown, then let ECS restart
    process.exit(1); // ECS will restart the task automatically
});

process.on('unhandledRejection', (reason, promise) => {
    logger.error('Unhandled Promise rejection', { reason, promise });
    // Log and continue — don't crash for unhandled rejections in non-critical paths
});

// Per-route error boundary
app.addHook('onError', (request, reply, error, done) => {
    logger.error('Request error', { path: request.url, error });
    reply.status(500).send({ error: 'Internal server error' });
    done();
});
```

ECS Fargate will restart the task if it crashes. With multi-AZ deployment, traffic is automatically routed to healthy tasks by the ALB.

---

### Limitation 3: Memory Leaks Are Common

**Problem:** JavaScript closures, event emitter listeners, and poorly managed caches cause gradual memory growth. Unlike Go's explicit ownership, JS makes it easy to accidentally hold references.

**Common leak sources:**

```javascript
// LEAK: Event listener added but never removed
eventEmitter.on('data', handler); // added on every request, never removed!

// LEAK: Map growing unbounded
const pendingRequests = new Map();
pendingRequests.set(requestId, callback); // never deleted if request times out

// LEAK: Closure holding reference to large object
function processLargeObject(obj) {
    const summary = extract(obj);  // extract what you need
    return () => console.log(summary); // this closure holds only `summary`
    // if it returned `obj` instead, the large object is never GC'd
}
```

**Fix:** Monitor heap usage, set memory limits in ECS, and restart on threshold.

```javascript
// Monitor memory every 30 seconds
setInterval(() => {
    const { heapUsed, heapTotal, rss } = process.memoryUsage();
    logger.info('Memory', {
        heapUsedMB: Math.round(heapUsed / 1024 / 1024),
        heapTotalMB: Math.round(heapTotal / 1024 / 1024),
        rssMB: Math.round(rss / 1024 / 1024)
    });
    
    // Alert if heap exceeds 80% of limit
    if (heapUsed / heapTotal > 0.8) {
        logger.warn('High memory usage — possible leak');
    }
}, 30000);
```

---

### Limitation 4: Cold Start Is Slower Than Go

**Problem:** Node.js takes 1-3 seconds to start (load modules, JIT compile, warmup). Go binary starts in <500ms.

**When it hurts:** ECS scale-out events — new tasks serve traffic slowly.

**Fix:** Keep a warm minimum instance count. ECS always has at least 2 tasks running. Scale-out adds a third (new task starts cold, but existing tasks absorb load during startup).

```javascript
// Signal readiness explicitly — ECS health check uses this
app.get('/health', async (req, res) => {
    if (!isWarmedUp) {
        return res.status(503).json({ status: 'starting' }); // ALB won't route to this yet
    }
    
    const redisOk = await redis.ping().then(() => true).catch(() => false);
    res.json({ status: 'ok', redis: redisOk });
});

// Warmup: pre-load caches before accepting traffic
async function warmup() {
    await loadOfferCache();        // MySQL → in-memory
    await verifyRedisConnection(); // Test Redis connectivity
    await workerPool.initialize(); // Spin up Worker threads
    isWarmedUp = true;             // Now ALB health check will return 200
    logger.info('LeadCore warmed up, accepting traffic');
}
```

---

### Limitation 5: Serialization Overhead for Worker Threads

**Problem:** Data passed between main thread and Worker threads is copied (serialized).

**Fix:** Keep data in Worker thread memory. Only send minimal data across the boundary.

```javascript
// SLOW: send 400KB of offers on every request
const results = await worker.postMessage({ offers: allOffers, profile }); // 400KB copied

// FAST: worker already has offers in its own memory
// Only send profile (1KB) + receive results (2KB)
const results = await workerPool.evaluateTargeting(profile); // <3KB total transfer
// Workers receive cache updates separately, every 60s, not per request
```

---

### Limitation 6: Callback Error Handling

**Problem:** Forgotten `.catch()` or missing `try/catch` around `await` silently swallows errors.

```javascript
// DANGEROUS: if redis.get throws, error is silently swallowed
async function loadData() {
    const data = await redis.get('key'); // if this throws, nothing happens
    return data;
}

// CORRECT: always handle errors
async function loadData() {
    try {
        const data = await redis.get('key');
        return data;
    } catch (error) {
        logger.error('Redis error in loadData', { error });
        return null; // graceful fallback
    }
}
```

---

## 12. Mental Models — How to Think About Node.js

### The Post Office Analogy

```
MULTI-THREADED SERVER (PHP-FPM):
  10 postal clerks (threads)
  Each clerk handles one customer end-to-end
  Clerk 1: "Hi! I'll get your package. Wait while I go to the back... [5 min]"
  Clerk 2: "Hi! I'll check your mailbox... [3 min]"
  Customer 11 WAITS in line — no clerks available

NODE.JS EVENT LOOP:
  1 postal clerk, but extremely organized
  "Next! What do you need? Package? I've sent the request to the back. [ticket 001]"
  "Next! Mailbox? Request sent. [ticket 002]"
  "Next! Address change? Done! Here you go."
  "Ticket 002 ready!" → calls customer 002, resolves their request
  "Ticket 001 ready!" → calls customer 001, resolves their request
  Nobody waits in line. Clerk is always working.
```

The clerk (Event Loop) is efficient because retrieval from the back (I/O) takes time but doesn't need the clerk's attention. The clerk's limitation: if a customer needs the clerk to do actual manual work (CPU), everyone else waits.

### The Three Questions to Ask About Any Node.js Code

Before writing any function, ask:

1. **Is this I/O-bound or CPU-bound?**
   - I/O: use `async/await`, it's fine on the Event Loop
   - CPU: use Worker Thread, or find a way to eliminate the CPU work

2. **Should these operations be sequential or concurrent?**
   - Sequential: `await a; await b;` — use when B depends on A's result
   - Concurrent: `await Promise.all([a, b])` — use when A and B are independent

3. **Does this need to block the response, or can it fire-and-forget?**
   - Blocks response: `await` it
   - Fire-and-forget: use `.catch()` without `await` for non-critical tracking/logging

### Quick Reference: When Node.js Wins vs Loses

```
NODE.JS WINS:
  ✓ High concurrency with I/O-bound work (API servers, proxies)
  ✓ Real-time event handling (WebSockets, SSE)
  ✓ Services with many small database/cache operations
  ✓ JSON API servers with simple business logic
  ✓ Proxy/gateway services (transform + forward requests)
  ✓ Streaming data pipelines

NODE.JS LOSES:
  ✗ CPU-heavy algorithms (ML inference, video encoding, crypto mining)
  ✗ Services that do heavy CPU work per request without Worker Threads
  ✗ Applications requiring strict memory isolation per request
  ✗ Long-running computations that can't be offloaded to workers
  ✗ Applications where GC pauses are unacceptable (real-time trading)

FOR LEADCORE SPECIFICALLY:
  ✓ Redis budget checks (I/O-bound — Node.js perfect)
  ✓ MySQL offer loading (I/O-bound — Node.js perfect)
  ✓ Kinesis event publishing (I/O-bound — Node.js perfect)
  ✓ Click/impression tracking (I/O-bound — Node.js perfect)
  ⚠ Targeting evaluation (CPU-bound — needs Worker Threads to scale)
  ✓ Response building (trivial CPU — Node.js fine)
```

---

## Summary

Node.js is a powerful choice for network-heavy services like LeadCore because:

1. **One thread handles thousands of concurrent I/O operations** via the non-blocking Event Loop
2. **`async/await` makes non-blocking code readable** without callback hell
3. **`Promise.all` gives you free concurrency** for independent I/O operations
4. **The npm ecosystem** has every integration you need (Redis, MySQL, Kinesis, AWS)

The key constraint to always remember:

> **The Event Loop is single-threaded. CPU work blocks everything. I/O work blocks nothing.**

Design for this constraint from day one:
- Identify every CPU-bound operation and move it to Worker Threads
- Use `Promise.all` for all independent I/O operations
- Fire-and-forget non-critical tracking events
- Monitor Event Loop lag — it tells you immediately if something is blocking

At our current traffic (25 req/sec), Node.js without Worker Threads would work fine. At 10× scale, Worker Threads become mandatory. Build it correctly from the start.

---

*Document: Node.js Internals — How It Really Works*
*Version: 1.0*
*Date: April 2026*
*Audience: Engineers building or reviewing the LeadCore Node.js service*
