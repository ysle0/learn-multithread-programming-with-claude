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

## Internal Mechanisms

### V8 Event Loop Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         V8 JavaScript Engine                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                      Heap (Object Storage)                     │ │
│  └───────────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                   Call Stack (Execution Context)               │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          libuv (Node.js)                            │
│                   또는 Browser Event Loop                            │
│                                                                     │
│  ┌─────────────────┐      ┌─────────────────────────────────────┐ │
│  │  Timer Phase    │      │  Microtask Queue                    │ │
│  │  (setTimeout)   │      │  [Promise.then] [queueMicrotask]    │ │
│  └────────┬────────┘      │  [MutationObserver]                 │ │
│           ▼               └─────────────────────────────────────┘ │
│  ┌─────────────────┐      ┌─────────────────────────────────────┐ │
│  │ Pending Callbacks│      │  Macrotask Queue                    │ │
│  │ (I/O callbacks) │      │  [setTimeout] [setInterval]         │ │
│  └────────┬────────┘      │  [setImmediate] [I/O]               │ │
│           ▼               └─────────────────────────────────────┘ │
│  ┌─────────────────┐                                              │
│  │    Poll Phase   │ ← I/O events (network, file, etc.)          │
│  └────────┬────────┘                                              │
│           ▼                                                        │
│  ┌─────────────────┐                                              │
│  │  Check Phase    │ ← setImmediate callbacks                     │
│  │ (setImmediate)  │                                              │
│  └────────┬────────┘                                              │
│           ▼                                                        │
│  ┌─────────────────┐                                              │
│  │  Close Phase    │ ← socket.on('close', ...)                    │
│  └─────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Loop Tick 상세

```javascript
// 각 단계에서 microtask queue 전체 처리
while (true) {
    // 1. 현재 phase의 macrotask 하나 실행
    executeMacrotask();

    // 2. microtask queue 비우기 (모두 실행)
    while (microtaskQueue.length > 0) {
        executeMicrotask(microtaskQueue.shift());

        // microtask가 새 microtask 추가하면 계속 실행!
        // → 무한 루프 가능
    }

    // 3. 렌더링 (브라우저에서만, 16ms마다)
    if (timeForRender) {
        requestAnimationFrame callbacks;
        render();
    }

    // 4. 다음 phase로
    moveToNextPhase();
}
```

### Node.js libuv Phases 상세

```
┌───────────────────────────────────────────────────────────────┐
│                     libuv Event Loop                          │
│                                                               │
│   ┌─────────┐                                                │
│   │  Timer  │ ← setTimeout, setInterval 만료된 것들           │
│   └────┬────┘                                                │
│        │ process.nextTick queue 처리                          │
│        │ microtask queue 처리                                 │
│        ▼                                                      │
│   ┌─────────┐                                                │
│   │ Pending │ ← 이전 iteration에서 지연된 I/O callbacks       │
│   └────┬────┘                                                │
│        │ process.nextTick queue 처리                          │
│        │ microtask queue 처리                                 │
│        ▼                                                      │
│   ┌─────────┐                                                │
│   │  Idle,  │ ← libuv 내부 사용                              │
│   │ Prepare │                                                │
│   └────┬────┘                                                │
│        ▼                                                      │
│   ┌─────────┐                                                │
│   │  Poll   │ ← I/O 이벤트 대기                              │
│   │         │   - 대기 시간: 다음 타이머까지 또는 무한        │
│   │         │   - 새 I/O 이벤트 수신 시 콜백 실행            │
│   └────┬────┘                                                │
│        │ process.nextTick queue 처리                          │
│        │ microtask queue 처리                                 │
│        ▼                                                      │
│   ┌─────────┐                                                │
│   │  Check  │ ← setImmediate callbacks                       │
│   └────┬────┘                                                │
│        │ process.nextTick queue 처리                          │
│        │ microtask queue 처리                                 │
│        ▼                                                      │
│   ┌─────────┐                                                │
│   │  Close  │ ← socket.on('close'), cleanup                  │
│   └─────────┘                                                │
│        │                                                      │
│        └──────────────────── loop ───────────────────────────▶│
└───────────────────────────────────────────────────────────────┘
```

### process.nextTick vs Promise vs setImmediate

```javascript
// 실행 순서:
// 1. 동기 코드
// 2. process.nextTick (Node.js 전용, 현재 phase 끝에 즉시)
// 3. Promise.then (microtask queue)
// 4. setImmediate (check phase)
// 5. setTimeout(..., 0) (timer phase, 다음 iteration)

console.log('1 sync');

process.nextTick(() => console.log('2 nextTick'));

Promise.resolve().then(() => console.log('3 Promise'));

setImmediate(() => console.log('4 setImmediate'));

setTimeout(() => console.log('5 setTimeout'), 0);

console.log('6 sync');

// 출력: 1 sync, 6 sync, 2 nextTick, 3 Promise, 4 setImmediate, 5 setTimeout
```

### 브라우저 렌더링과 Event Loop

```
1 frame (약 16.67ms for 60fps)
┌────────────────────────────────────────────────────────────┐
│ JavaScript │ Style │ Layout │ Paint │ Composite │ Idle   │
│ (macrotask)│ Calc  │        │       │           │        │
└────────────────────────────────────────────────────────────┘

requestAnimationFrame은 Paint 직전에 실행:
- Layout 계산 후, Paint 전
- 애니메이션에 이상적

requestIdleCallback은 Idle 시간에 실행:
- 우선순위 낮은 작업용
- 프레임에 여유 시간 있을 때만
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
