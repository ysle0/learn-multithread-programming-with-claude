# JavaScript Concurrency

JavaScript uses an event-driven, single-threaded concurrency model with async/await for I/O operations and Web Workers for true parallelism. Understanding the event loop is key to mastering JavaScript concurrency.

## Overview

JavaScript's concurrency evolved through several phases:
- **Callbacks**: Original async pattern (callback hell)
- **Promises (ES6)**: Chainable async operations
- **Async/Await (ES2017)**: Syntactic sugar over promises
- **Web Workers**: True multi-threading in browsers
- **Worker Threads (Node.js)**: Multi-threading on server
- **SharedArrayBuffer**: Shared memory between workers

## Core Components

### 1. [Event Loop](./01-event-loop.md)
- Call stack and task queue
- Microtasks vs. macrotasks
- Event loop phases
- Understanding non-blocking I/O

### 2. [Async/Await](./02-async-await.md)
- Promise-based async programming
- async function declarations
- await expressions
- Error handling with try/catch

### 3. [Web Workers](./03-web-workers.md)
- Browser-based multi-threading
- Worker creation and messaging
- Transferable objects
- Worker lifecycle

### 4. [Worker Threads (Node.js)](./04-worker-threads.md)
- Server-side multi-threading
- Worker thread API
- Sharing data between threads
- Thread pools

### 5. [SharedArrayBuffer](./05-shared-array-buffer.md)
- Shared memory between workers
- Atomics for synchronization
- Race condition prevention
- Futex-style primitives

## Quick Comparison with Other Languages

| Feature | JavaScript | Comparison |
|---------|-----------|------------|
| **Main Model** | Event loop + async/await | Best for I/O, weakest for CPU |
| **True Threading** | Workers (heavy) | Most limited threading model |
| **Async Syntax** | `async/await` | Cleanest syntax (with C#) |
| **Default Behavior** | Non-blocking I/O | Prevents thread blocking naturally |
| **Shared Memory** | Limited (SharedArrayBuffer) | Most restricted |

## Key Principles

### 1. Single-Threaded Main Execution

```javascript
// Main thread is single-threaded
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');
// Prints: 1, 3, 2
```

### 2. Non-Blocking I/O

```javascript
// Doesn't block main thread
async function fetchData() {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
}
```

### 3. Event Loop Drives Execution

```javascript
// Event loop processes tasks from queue
Promise.resolve().then(() => console.log('microtask'));
setTimeout(() => console.log('macrotask'), 0);
console.log('synchronous');
// Prints: synchronous, microtask, macrotask
```

## Common Patterns

### Parallel Async Operations

```javascript
// Sequential (slow)
const data1 = await fetch('/api/1');
const data2 = await fetch('/api/2');

// Parallel (fast)
const [data1, data2] = await Promise.all([
    fetch('/api/1'),
    fetch('/api/2')
]);
```

### Worker Communication

```javascript
// Main thread
const worker = new Worker('worker.js');
worker.postMessage({ task: 'compute', data: [1, 2, 3] });
worker.onmessage = (e) => {
    console.log('Result:', e.data);
};

// worker.js
self.onmessage = (e) => {
    const result = compute(e.data.data);
    self.postMessage(result);
};
```

## Best Practices

1. **Use async/await over callbacks**: Cleaner, more readable
2. **Don't block the event loop**: Use Workers for CPU-intensive tasks
3. **Use Promise.all for parallel operations**: Don't await sequentially
4. **Handle Promise rejections**: Always catch errors
5. **Prefer Workers over synchronous operations**: For heavy computation
6. **Use AbortController for cancellation**: Standard cancellation API
7. **Avoid SharedArrayBuffer unless necessary**: Complex and error-prone

## Common Pitfalls

### 1. Blocking the Event Loop

```javascript
// BAD: Blocks entire application
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
const result = fibonacci(45);  // Freezes app!

// GOOD: Use Worker
const worker = new Worker('fibonacci-worker.js');
worker.postMessage(45);
worker.onmessage = (e) => console.log('Result:', e.data);
```

### 2. Sequential Instead of Parallel

```javascript
// BAD: Sequential (slow)
async function fetchAll() {
    const user = await fetchUser();
    const posts = await fetchPosts();
    const comments = await fetchComments();
    return { user, posts, comments };
}

// GOOD: Parallel (fast)
async function fetchAll() {
    const [user, posts, comments] = await Promise.all([
        fetchUser(),
        fetchPosts(),
        fetchComments()
    ]);
    return { user, posts, comments };
}
```

### 3. Unhandled Promise Rejections

```javascript
// BAD: Unhandled rejection
async function badFetch() {
    const data = await fetch('/api/data');
    // If fetch fails, error propagates up
}

// GOOD: Handle errors
async function goodFetch() {
    try {
        const data = await fetch('/api/data');
        return data;
    } catch (error) {
        console.error('Fetch failed:', error);
        return null;
    }
}
```

### 4. Not Cleaning Up Workers

```javascript
// BAD: Worker leak
function createWorker() {
    const worker = new Worker('worker.js');
    // Worker never terminated
}

// GOOD: Terminate when done
function createWorker() {
    const worker = new Worker('worker.js');
    worker.onmessage = (e) => {
        console.log(e.data);
        worker.terminate();  // Clean up
    };
}
```

### 5. Race Conditions with SharedArrayBuffer

```javascript
// BAD: Data race
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

// Multiple workers:
view[0]++;  // Not atomic!

// GOOD: Use Atomics
Atomics.add(view, 0, 1);
```

## Performance Considerations

### Event Loop Blocking
- Keep tasks < 16ms for 60 FPS
- Use `setTimeout(..., 0)` to yield control
- Workers for CPU-intensive tasks

### Promise Overhead
- ~100ns per Promise
- Negligible for I/O-bound operations
- Consider for micro-optimizations

### Worker Overhead
- **Creation**: ~10-50ms
- **Message passing**: Copy overhead (use transferables)
- **Communication**: Slower than threads in other languages

## Environment Differences

### Browser vs. Node.js

```javascript
// Browser: Web Workers
const worker = new Worker('worker.js');

// Node.js: Worker Threads
const { Worker } = require('worker_threads');
const worker = new Worker('./worker.js');
```

### Feature Detection

```javascript
if (typeof Worker !== 'undefined') {
    // Web Workers available
}

if (typeof SharedArrayBuffer !== 'undefined') {
    // Shared memory available
}
```

## Modern JavaScript Features

### Async Iterators

```javascript
async function* asyncGenerator() {
    for (let i = 0; i < 5; i++) {
        await new Promise(resolve => setTimeout(resolve, 100));
        yield i;
    }
}

for await (const value of asyncGenerator()) {
    console.log(value);
}
```

### Top-Level Await (ES2022)

```javascript
// In modules
const data = await fetch('/api/data');
console.log(data);
```

### AbortController

```javascript
const controller = new AbortController();
const signal = controller.signal;

fetch('/api/data', { signal })
    .then(response => response.json())
    .catch(err => {
        if (err.name === 'AbortError') {
            console.log('Fetch aborted');
        }
    });

// Abort after 5 seconds
setTimeout(() => controller.abort(), 5000);
```

## Tools and Debugging

### Chrome DevTools
- Performance tab for event loop analysis
- Memory profiler for Worker leaks
- Console for async stack traces

### Node.js Profiling
```bash
node --inspect app.js
# Open chrome://inspect
```

### Performance Monitoring

```javascript
console.time('operation');
await expensiveOperation();
console.timeEnd('operation');

// Or use Performance API
const start = performance.now();
await expensiveOperation();
const end = performance.now();
console.log(`Took ${end - start}ms`);
```

## Further Reading

- **Eloquent JavaScript** by Marijn Haverbeke
- **You Don't Know JS: Async & Performance** by Kyle Simpson
- MDN Web Docs: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide
- Jake Archibald's Event Loop Talk

## Navigation

- [Back to Language Implementations](../)
- Next Topics:
  - [Event Loop](./01-event-loop.md)
  - [Async/Await](./02-async-await.md)
  - [Web Workers](./03-web-workers.md)
  - [Worker Threads](./04-worker-threads.md)
  - [SharedArrayBuffer](./05-shared-array-buffer.md)
