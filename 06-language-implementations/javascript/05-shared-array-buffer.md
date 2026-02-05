# JavaScript의 SharedArrayBuffer와 Atomics

SharedArrayBuffer는 Worker 간 진정한 공유 메모리를 가능하게 하며, Atomics는 동기화 프리미티브를 제공합니다.

## 기본 공유 메모리

### SharedArrayBuffer 생성

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

    // 양쪽 모두 같은 메모리에 접근 가능
    sharedArray[0] = 42;
};
```

## Atomics

### 원자적 연산

```javascript
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

// 원자적 연산
Atomics.add(view, 0, 5);        // 인덱스 0에 5 더하기
Atomics.sub(view, 0, 2);        // 2 빼기
Atomics.load(view, 0);          // 값 읽기
Atomics.store(view, 0, 10);     // 값 쓰기
Atomics.exchange(view, 0, 20);  // 값 교환

// 비교 후 교환
const oldValue = Atomics.compareExchange(view, 0, 20, 30);
// view[0] === 20이면, 30으로 설정하고 20 반환
// 그렇지 않으면 현재 값 반환
```

### Wait와 Notify (Futex)

```javascript
// main.js
const sab = new SharedArrayBuffer(4);
const view = new Int32Array(sab);

const worker = new Worker('worker.js');
worker.postMessage(sab);

setTimeout(() => {
    Atomics.store(view, 0, 1);
    Atomics.notify(view, 0, 1);  // 대기 중인 하나를 깨움
}, 1000);

// worker.js
self.onmessage = (event) => {
    const view = new Int32Array(event.data);

    console.log('대기 중...');
    Atomics.wait(view, 0, 0);  // view[0] !== 0이 될 때까지 대기
    console.log('깨어남!');
};
```

## 동기화 패턴

### Spinlock

```javascript
class Spinlock {
    constructor(sharedBuffer, index) {
        this.view = new Int32Array(sharedBuffer);
        this.index = index;
    }

    lock() {
        while (Atomics.compareExchange(this.view, this.index, 0, 1) !== 0) {
            // 스핀
        }
    }

    unlock() {
        Atomics.store(this.view, this.index, 0);
    }
}

// 사용법
const sab = new SharedArrayBuffer(4);
const lock = new Spinlock(sab, 0);

lock.lock();
try {
    // 임계 영역
} finally {
    lock.unlock();
}
```

### Wait/Notify를 이용한 Mutex

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
                return;  // 잠금 획득
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

## 완전한 예제: 생산자-소비자

```javascript
// 공유 상태
const BUFFER_SIZE = 10;
const STATE_SIZE = 3; // [lock, count, closed]
const TOTAL_SIZE = STATE_SIZE + BUFFER_SIZE;

const sab = new SharedArrayBuffer(TOTAL_SIZE * 4);
const state = new Int32Array(sab);

// 생산자
const producer = new Worker('producer.js');
producer.postMessage(sab);

// 소비자
const consumer = new Worker('consumer.js');
consumer.postMessage(sab);

// producer.js
self.onmessage = (event) => {
    const state = new Int32Array(event.data);

    for (let i = 0; i < 20; i++) {
        // 공간이 생길 때까지 대기
        while (Atomics.load(state, 1) >= 10) {
            Atomics.wait(state, 1, 10);
        }

        // 항목 추가
        const count = Atomics.load(state, 1);
        Atomics.store(state, 3 + count, i);
        Atomics.add(state, 1, 1);
        Atomics.notify(state, 1, 1);
    }

    // 완료 신호
    Atomics.store(state, 2, 1);
    Atomics.notify(state, 1, Infinity);
};

// consumer.js
self.onmessage = (event) => {
    const state = new Int32Array(event.data);

    while (true) {
        // 항목이 생길 때까지 대기
        while (Atomics.load(state, 1) === 0 && Atomics.load(state, 2) === 0) {
            Atomics.wait(state, 1, 0);
        }

        if (Atomics.load(state, 1) === 0 && Atomics.load(state, 2) === 1) {
            break;  // 완료
        }

        // 항목 가져오기
        const count = Atomics.load(state, 1);
        const item = Atomics.load(state, 3 + count - 1);
        console.log('소비:', item);
        Atomics.sub(state, 1, 1);
        Atomics.notify(state, 1, 1);
    }
};
```

## 내부 메커니즘

### SharedArrayBuffer 메모리 모델

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SharedArrayBuffer                                │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              공유 메모리 영역 (mmap)                          │ │
│  │  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                   │ │
│  │  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │...│ n │  bytes            │ │
│  │  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                   │ │
│  └───────────────────────────────────────────────────────────────┘ │
│         ▲                                         ▲                │
│         │                                         │                │
│  ┌──────┴──────┐                          ┌──────┴──────┐         │
│  │ TypedArray  │                          │ TypedArray  │         │
│  │ (Worker 1)  │                          │ (Worker 2)  │         │
│  │ Int32Array  │                          │ Int32Array  │         │
│  └─────────────┘                          └─────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

### Atomics.wait/notify (Futex 스타일)

```javascript
// Atomics.wait는 Linux futex와 유사하게 구현됨
// 값이 expected와 같으면 sleep, 다르면 즉시 반환

Atomics.wait(view, index, expectedValue, timeout);
// 반환값: "ok" (깨어남), "timed-out", "not-equal"

// Main thread에서는 wait 불가 (브라우저)
// Node.js에서는 Atomics.wait.sync 없이 직접 사용

// Atomics.notify는 대기 중인 worker 깨움
Atomics.notify(view, index, count);
// count: 깨울 worker 수 (Infinity = 모두)
```

### Spectre 취약점과 보안

```
Spectre 공격:
- SharedArrayBuffer + 고해상도 타이머로 캐시 타이밍 공격 가능
- 2018년 대부분 브라우저에서 비활성화

해결책 (Cross-Origin Isolation):
- COOP: Cross-Origin-Opener-Policy: same-origin
  → 새 browsing context group 생성
- COEP: Cross-Origin-Embedder-Policy: require-corp
  → 모든 리소스가 CORP 헤더 필요

// 격리 확인
if (crossOriginIsolated) {
    const sab = new SharedArrayBuffer(1024);  // OK
}
```

## 보안 고려사항

SharedArrayBuffer를 사용하려면 특정 헤더가 필요합니다:
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## 내비게이션

- [JavaScript 개요로 돌아가기](./README.md)
- 이전: [Worker Threads](./04-worker-threads.md)
- [언어 구현으로 돌아가기](../)
