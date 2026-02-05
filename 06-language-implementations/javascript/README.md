# JavaScript 동시성

JavaScript는 이벤트 기반 단일 스레드 동시성 모델을 사용하며, I/O 작업을 위한 async/await와 진정한 병렬 처리를 위한 Web Worker를 제공합니다. Event loop를 이해하는 것이 JavaScript 동시성을 마스터하는 핵심입니다.

## 개요

JavaScript의 동시성은 여러 단계를 거쳐 발전해 왔습니다:
- **Callbacks**: 최초의 비동기 패턴 (콜백 지옥)
- **Promises (ES6)**: 체이닝 가능한 비동기 연산
- **Async/Await (ES2017)**: Promise 위에 구축된 문법적 설탕
- **Web Workers**: 브라우저에서의 진정한 멀티스레딩
- **Worker Threads (Node.js)**: 서버 측 멀티스레딩
- **SharedArrayBuffer**: Worker 간 공유 메모리

## 핵심 구성 요소

### 1. [Event Loop](./01-event-loop.md)
- Call stack과 task queue
- Microtask와 macrotask의 차이
- Event loop 단계
- 논블로킹 I/O 이해하기

### 2. [Async/Await](./02-async-await.md)
- Promise 기반 비동기 프로그래밍
- async 함수 선언
- await 표현식
- try/catch를 이용한 에러 처리

### 3. [Web Workers](./03-web-workers.md)
- 브라우저 기반 멀티스레딩
- Worker 생성 및 메시징
- Transferable objects
- Worker 생명주기

### 4. [Worker Threads (Node.js)](./04-worker-threads.md)
- 서버 측 멀티스레딩
- Worker thread API
- 스레드 간 데이터 공유
- 스레드 풀

### 5. [SharedArrayBuffer](./05-shared-array-buffer.md)
- Worker 간 공유 메모리
- 동기화를 위한 Atomics
- 경쟁 조건 방지
- Futex 스타일 프리미티브

## 다른 언어와의 간략한 비교

| 특성 | JavaScript | 비교 |
|---------|-----------|------------|
| **주요 모델** | Event loop + async/await | I/O에 최적, CPU 처리에 가장 약함 |
| **진정한 스레딩** | Workers (무거움) | 가장 제한적인 스레딩 모델 |
| **비동기 문법** | `async/await` | 가장 깔끔한 문법 (C#과 함께) |
| **기본 동작** | 논블로킹 I/O | 자연스럽게 스레드 블로킹 방지 |
| **공유 메모리** | 제한적 (SharedArrayBuffer) | 가장 제한적 |

## 핵심 원칙

### 1. 단일 스레드 메인 실행

```javascript
// 메인 스레드는 단일 스레드
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');
// 출력: 1, 3, 2
```

### 2. 논블로킹 I/O

```javascript
// 메인 스레드를 블로킹하지 않음
async function fetchData() {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
}
```

### 3. Event Loop가 실행을 주도

```javascript
// Event loop가 큐에서 작업을 처리
Promise.resolve().then(() => console.log('microtask'));
setTimeout(() => console.log('macrotask'), 0);
console.log('synchronous');
// 출력: synchronous, microtask, macrotask
```

## 일반적인 패턴

### 병렬 비동기 연산

```javascript
// 순차적 (느림)
const data1 = await fetch('/api/1');
const data2 = await fetch('/api/2');

// 병렬 (빠름)
const [data1, data2] = await Promise.all([
    fetch('/api/1'),
    fetch('/api/2')
]);
```

### Worker 통신

```javascript
// 메인 스레드
const worker = new Worker('worker.js');
worker.postMessage({ task: 'compute', data: [1, 2, 3] });
worker.onmessage = (e) => {
    console.log('결과:', e.data);
};

// worker.js
self.onmessage = (e) => {
    const result = compute(e.data.data);
    self.postMessage(result);
};
```

## 모범 사례

1. **콜백 대신 async/await 사용**: 더 깔끔하고 가독성 높음
2. **Event loop를 블로킹하지 말 것**: CPU 집약적 작업에는 Worker 사용
3. **병렬 연산에는 Promise.all 사용**: 순차적으로 await하지 말 것
4. **Promise 거부 처리하기**: 항상 에러를 잡을 것
5. **동기 연산보다 Workers 선호**: 무거운 계산 작업용
6. **취소에는 AbortController 사용**: 표준 취소 API
7. **필요하지 않으면 SharedArrayBuffer 사용 자제**: 복잡하고 에러 발생 가능성 높음

## 흔한 실수

### 1. Event Loop 블로킹

```javascript
// 나쁜 예: 전체 애플리케이션을 블로킹
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
const result = fibonacci(45);  // 앱이 멈춤!

// 좋은 예: Worker 사용
const worker = new Worker('fibonacci-worker.js');
worker.postMessage(45);
worker.onmessage = (e) => console.log('결과:', e.data);
```

### 2. 병렬 대신 순차 실행

```javascript
// 나쁜 예: 순차적 (느림)
async function fetchAll() {
    const user = await fetchUser();
    const posts = await fetchPosts();
    const comments = await fetchComments();
    return { user, posts, comments };
}

// 좋은 예: 병렬 (빠름)
async function fetchAll() {
    const [user, posts, comments] = await Promise.all([
        fetchUser(),
        fetchPosts(),
        fetchComments()
    ]);
    return { user, posts, comments };
}
```

### 3. 처리되지 않은 Promise 거부

```javascript
// 나쁜 예: 처리되지 않은 거부
async function badFetch() {
    const data = await fetch('/api/data');
    // fetch가 실패하면 에러가 상위로 전파됨
}

// 좋은 예: 에러 처리
async function goodFetch() {
    try {
        const data = await fetch('/api/data');
        return data;
    } catch (error) {
        console.error('Fetch 실패:', error);
        return null;
    }
}
```

### 4. Worker 정리를 하지 않음

```javascript
// 나쁜 예: Worker 누수
function createWorker() {
    const worker = new Worker('worker.js');
    // Worker가 종료되지 않음
}

// 좋은 예: 완료 시 종료
function createWorker() {
    const worker = new Worker('worker.js');
    worker.onmessage = (e) => {
        console.log(e.data);
        worker.terminate();  // 정리
    };
}
```

### 5. SharedArrayBuffer 경쟁 조건

```javascript
// 나쁜 예: 데이터 경쟁
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

// 여러 worker에서:
view[0]++;  // 원자적이지 않음!

// 좋은 예: Atomics 사용
Atomics.add(view, 0, 1);
```

## 성능 고려사항

### Event Loop 블로킹
- 60 FPS를 위해 작업을 16ms 미만으로 유지
- `setTimeout(..., 0)`으로 제어권 양보
- CPU 집약적 작업에는 Workers 사용

### Promise 오버헤드
- Promise당 약 100ns
- I/O 바운드 연산에서는 무시할 수 있는 수준
- 마이크로 최적화에서 고려

### Worker 오버헤드
- **생성**: 약 10-50ms
- **메시지 전달**: 복사 오버헤드 (transferable 사용 권장)
- **통신**: 다른 언어의 스레드보다 느림

## 환경 차이

### 브라우저 vs. Node.js

```javascript
// 브라우저: Web Workers
const worker = new Worker('worker.js');

// Node.js: Worker Threads
const { Worker } = require('worker_threads');
const worker = new Worker('./worker.js');
```

### 기능 감지

```javascript
if (typeof Worker !== 'undefined') {
    // Web Workers 사용 가능
}

if (typeof SharedArrayBuffer !== 'undefined') {
    // 공유 메모리 사용 가능
}
```

## 최신 JavaScript 기능

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
// 모듈에서
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
            console.log('Fetch가 중단됨');
        }
    });

// 5초 후 중단
setTimeout(() => controller.abort(), 5000);
```

## 도구 및 디버깅

### Chrome DevTools
- Event loop 분석을 위한 Performance 탭
- Worker 누수 확인을 위한 메모리 프로파일러
- 비동기 스택 트레이스를 위한 콘솔

### Node.js 프로파일링
```bash
node --inspect app.js
# chrome://inspect 열기
```

### 성능 모니터링

```javascript
console.time('operation');
await expensiveOperation();
console.timeEnd('operation');

// 또는 Performance API 사용
const start = performance.now();
await expensiveOperation();
const end = performance.now();
console.log(`소요 시간: ${end - start}ms`);
```

## 추가 읽을거리

- **Eloquent JavaScript** by Marijn Haverbeke
- **You Don't Know JS: Async & Performance** by Kyle Simpson
- MDN Web Docs: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide
- Jake Archibald의 Event Loop 발표

## 내비게이션

- [언어 구현으로 돌아가기](../)
- 다음 주제:
  - [Event Loop](./01-event-loop.md)
  - [Async/Await](./02-async-await.md)
  - [Web Workers](./03-web-workers.md)
  - [Worker Threads](./04-worker-threads.md)
  - [SharedArrayBuffer](./05-shared-array-buffer.md)
