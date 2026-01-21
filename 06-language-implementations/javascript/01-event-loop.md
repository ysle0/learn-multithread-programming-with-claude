# Event Loop in JavaScript

The event loop is the heart of JavaScript's concurrency model, enabling non-blocking I/O despite being single-threaded.

## Table of Contents
- [How It Works](#how-it-works)
- [Call Stack](#call-stack)
- [Task Queue](#task-queue)
- [Microtasks vs Macrotasks](#microtasks-vs-macrotasks)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## How It Works

### Basic Model

```javascript
console.log('1');

setTimeout(() => {
    console.log('2');
}, 0);

Promise.resolve().then(() => {
    console.log('3');
});

console.log('4');

// Output: 1, 4, 3, 2
```

### Event Loop Phases

1. **Execute synchronous code**
2. **Process microtask queue** (Promises, queueMicrotask)
3. **Process macrotask queue** (setTimeout, setInterval, I/O)
4. **Repeat**

## Call Stack

```javascript
function first() {
    console.log('first');
    second();
}

function second() {
    console.log('second');
    third();
}

function third() {
    console.log('third');
}

first();
// Call stack: first -> second -> third
```

## Task Queue

### Macrotasks

```javascript
// setTimeout creates macrotask
setTimeout(() => {
    console.log('macrotask 1');
}, 0);

setTimeout(() => {
    console.log('macrotask 2');
}, 0);

// Prints: macrotask 1, macrotask 2
```

## Microtasks vs Macrotasks

### Microtasks (Priority)

```javascript
// Microtasks run BEFORE next macrotask
console.log('script start');

setTimeout(() => {
    console.log('setTimeout');
}, 0);

Promise.resolve()
    .then(() => {
        console.log('promise1');
    })
    .then(() => {
        console.log('promise2');
    });

console.log('script end');

// Output:
// script start
// script end
// promise1
// promise2
// setTimeout
```

### Execution Order

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

queueMicrotask(() => console.log('4'));

console.log('5');

// Output: 1, 5, 3, 4, 2
```

## Best Practices

### 1. Don't Block the Event Loop

```javascript
// BAD: Blocks for seconds
function blockingOperation() {
    const start = Date.now();
    while (Date.now() - start < 5000) {
        // Blocks!
    }
}

// GOOD: Break into chunks
async function nonBlockingOperation() {
    for (let i = 0; i < 100; i++) {
        doWork();
        await new Promise(resolve => setTimeout(resolve, 0));
    }
}
```

### 2. Use Microtasks for High Priority

```javascript
// High priority
queueMicrotask(() => {
    console.log('High priority');
});

// Lower priority
setTimeout(() => {
    console.log('Lower priority');
}, 0);
```

## Common Pitfalls

### Infinite Microtask Loop

```javascript
// BAD: Never gives macrotasks a chance
function infiniteMicrotasks() {
    Promise.resolve().then(() => {
        infiniteMicrotasks();
    });
}

infiniteMicrotasks();
// setTimeout never runs!
```

## Complete Example

```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout 1');
    Promise.resolve().then(() => console.log('Promise in timeout'));
}, 0);

setTimeout(() => {
    console.log('Timeout 2');
}, 0);

Promise.resolve()
    .then(() => {
        console.log('Promise 1');
    })
    .then(() => {
        console.log('Promise 2');
    });

console.log('End');

// Output:
// Start
// End
// Promise 1
// Promise 2
// Timeout 1
// Promise in timeout
// Timeout 2
```

## Navigation

- [Back to JavaScript Overview](./README.md)
- Next: [Async/Await](./02-async-await.md)
