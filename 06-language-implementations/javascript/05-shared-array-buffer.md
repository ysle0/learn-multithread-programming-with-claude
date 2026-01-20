# SharedArrayBuffer and Atomics in JavaScript

SharedArrayBuffer enables true shared memory between workers, with Atomics providing synchronization primitives.

## Basic Shared Memory

### Creating SharedArrayBuffer

```javascript
// main.js
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker('worker.js');
worker.postMessage(sharedBuffer);

// worker.js
self.onmessage = (event) => {
    const sharedBuffer = event.data;
    const sharedArray = new Int32Array(sharedBuffer);

    // Both can access same memory
    sharedArray[0] = 42;
};
```

## Atomics

### Atomic Operations

```javascript
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

// Atomic operations
Atomics.add(view, 0, 5);        // Add 5 to index 0
Atomics.sub(view, 0, 2);        // Subtract 2
Atomics.load(view, 0);          // Read value
Atomics.store(view, 0, 10);     // Write value
Atomics.exchange(view, 0, 20);  // Swap value

// Compare and exchange
const oldValue = Atomics.compareExchange(view, 0, 20, 30);
// If view[0] === 20, set to 30 and return 20
// Else return current value
```

### Wait and Notify (Futex)

```javascript
// main.js
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

const worker = new Worker('worker.js');
worker.postMessage(sab);

setTimeout(() => {
    Atomics.store(view, 0, 1);
    Atomics.notify(view, 0, 1);  // Wake one waiter
}, 1000);

// worker.js
self.onmessage = (event) => {
    const view = new Int32Array(event.data);

    console.log('Waiting...');
    Atomics.wait(view, 0, 0);  // Wait until view[0] !== 0
    console.log('Notified!');
};
```

## Synchronization Patterns

### Spinlock

```javascript
class Spinlock {
    constructor(sharedBuffer, index) {
        this.view = new Int32Array(sharedBuffer);
        this.index = index;
    }

    lock() {
        while (Atomics.compareExchange(this.view, this.index, 0, 1) !== 0) {
            // Spin
        }
    }

    unlock() {
        Atomics.store(this.view, this.index, 0);
    }
}

// Usage
const sab = new SharedArrayBuffer(4);
const lock = new Spinlock(sab, 0);

lock.lock();
try {
    // Critical section
} finally {
    lock.unlock();
}
```

### Mutex with Wait/Notify

```javascript
class Mutex {
    constructor(sharedBuffer, index) {
        this.view = new Int32Array(sharedBuffer);
        this.index = index;
    }

    lock() {
        while (true) {
            const old = Atomics.compareExchange(this.view, this.index, 0, 1);
            if (old === 0) {
                return;  // Acquired lock
            }
            Atomics.wait(this.view, this.index, 1);
        }
    }

    unlock() {
        Atomics.store(this.view, this.index, 0);
        Atomics.notify(this.view, this.index, 1);
    }
}
```

## Complete Example: Producer-Consumer

```javascript
// Shared state
const BUFFER_SIZE = 10;
const STATE_SIZE = 3; // [lock, count, closed]
const TOTAL_SIZE = STATE_SIZE + BUFFER_SIZE;

const sab = new SharedArrayBuffer(TOTAL_SIZE * 4);
const state = new Int32Array(sab);

// Producer
const producer = new Worker('producer.js');
producer.postMessage(sab);

// Consumer
const consumer = new Worker('consumer.js');
consumer.postMessage(sab);

// producer.js
self.onmessage = (event) => {
    const state = new Int32Array(event.data);

    for (let i = 0; i < 20; i++) {
        // Wait for space
        while (Atomics.load(state, 1) >= 10) {
            Atomics.wait(state, 1, 10);
        }

        // Add item
        const count = Atomics.load(state, 1);
        Atomics.store(state, 3 + count, i);
        Atomics.add(state, 1, 1);
        Atomics.notify(state, 1, 1);
    }

    // Signal done
    Atomics.store(state, 2, 1);
    Atomics.notify(state, 1, Infinity);
};

// consumer.js
self.onmessage = (event) => {
    const state = new Int32Array(event.data);

    while (true) {
        // Wait for items
        while (Atomics.load(state, 1) === 0 && Atomics.load(state, 2) === 0) {
            Atomics.wait(state, 1, 0);
        }

        if (Atomics.load(state, 1) === 0 && Atomics.load(state, 2) === 1) {
            break;  // Done
        }

        // Get item
        const count = Atomics.load(state, 1);
        const item = Atomics.load(state, 3 + count - 1);
        console.log('Consumed:', item);
        Atomics.sub(state, 1, 1);
        Atomics.notify(state, 1, 1);
    }
};
```

## Security Considerations

SharedArrayBuffer requires specific headers:
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## Navigation

- [Back to JavaScript Overview](./README.md)
- Previous: [Worker Threads](./04-worker-threads.md)
- [Back to Language Implementations](../)
