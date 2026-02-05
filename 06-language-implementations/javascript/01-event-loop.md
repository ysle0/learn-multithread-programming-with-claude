# JavaScript의 Event Loop

Event loop는 JavaScript 동시성 모델의 핵심으로, 단일 스레드임에도 불구하고 논블로킹 I/O를 가능하게 합니다.

## 목차
- [동작 원리](#동작-원리)
- [Call Stack](#call-stack)
- [Task Queue](#task-queue)
- [Microtask vs Macrotask](#microtask-vs-macrotask)
- [모범 사례](#모범-사례)
- [흔한 실수](#흔한-실수)

## 동작 원리

### 기본 모델

```javascript
console.log('1');

setTimeout(() => {
    console.log('2');
}, 0);

Promise.resolve().then(() => {
    console.log('3');
});

console.log('4');

// 출력: 1, 4, 3, 2
```

### Event Loop 단계

1. **동기 코드 실행**
2. **Microtask queue 처리** (Promises, queueMicrotask)
3. **Macrotask queue 처리** (setTimeout, setInterval, I/O)
4. **반복**

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

### Macrotask

```javascript
// setTimeout은 macrotask를 생성
setTimeout(() => {
    console.log('macrotask 1');
}, 0);

setTimeout(() => {
    console.log('macrotask 2');
}, 0);

// 출력: macrotask 1, macrotask 2
```

## Microtask vs Macrotask

### Microtask (우선순위)

```javascript
// Microtask는 다음 macrotask 이전에 실행됨
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

// 출력:
// script start
// script end
// promise1
// promise2
// setTimeout
```

### 실행 순서

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

queueMicrotask(() => console.log('4'));

console.log('5');

// 출력: 1, 5, 3, 4, 2
```

## 모범 사례

### 1. Event Loop를 블로킹하지 말 것

```javascript
// 나쁜 예: 수 초간 블로킹
function blockingOperation() {
    const start = Date.now();
    while (Date.now() - start < 5000) {
        // 블로킹!
    }
}

// 좋은 예: 청크로 분할
async function nonBlockingOperation() {
    for (let i = 0; i < 100; i++) {
        doWork();
        await new Promise(resolve => setTimeout(resolve, 0));
    }
}
```

### 2. 높은 우선순위에는 Microtask 사용

```javascript
// 높은 우선순위
queueMicrotask(() => {
    console.log('높은 우선순위');
});

// 낮은 우선순위
setTimeout(() => {
    console.log('낮은 우선순위');
}, 0);
```

## 흔한 실수

### 무한 Microtask 루프

```javascript
// 나쁜 예: macrotask에 기회를 주지 않음
function infiniteMicrotasks() {
    Promise.resolve().then(() => {
        infiniteMicrotasks();
    });
}

infiniteMicrotasks();
// setTimeout은 절대 실행되지 않음!
```

## 내부 메커니즘

### V8 Event Loop 아키텍처

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
│  │    Poll Phase   │ ← I/O 이벤트 (네트워크, 파일 등)            │
│  └────────┬────────┘                                              │
│           ▼                                                        │
│  ┌─────────────────┐                                              │
│  │  Check Phase    │ ← setImmediate 콜백                          │
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

## 완전한 예제

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

// 출력:
// Start
// End
// Promise 1
// Promise 2
// Timeout 1
// Promise in timeout
// Timeout 2
```

## 내비게이션

- [JavaScript 개요로 돌아가기](./README.md)
- 다음: [Async/Await](./02-async-await.md)
