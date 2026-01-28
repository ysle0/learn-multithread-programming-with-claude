# Async/Await in JavaScript

Async/await provides a clean, synchronous-looking syntax for asynchronous operations built on top of Promises.

## Basic Concepts

### Async Functions

```javascript
// Async function always returns a Promise
async function fetchData() {
    return 'data';
}

fetchData().then(data => console.log(data));  // 'data'

// Equivalent to:
function fetchDataOld() {
    return Promise.resolve('data');
}
```

### Await Expression

```javascript
async function getData() {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
}

// Equivalent to:
function getDataOld() {
    return fetch('https://api.example.com/data')
        .then(response => response.json());
}
```

## Error Handling

### Try-Catch

```javascript
async function fetchWithErrorHandling() {
    try {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error('HTTP error');
        }
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('Error:', error);
        return null;
    }
}
```

## Parallel Operations

### Promise.all

```javascript
async function fetchMultiple() {
    // Sequential (slow) - 6 seconds total
    const user = await fetch('/api/user');
    const posts = await fetch('/api/posts');
    const comments = await fetch('/api/comments');

    // Parallel (fast) - 2 seconds total
    const [userRes, postsRes, commentsRes] = await Promise.all([
        fetch('/api/user'),
        fetch('/api/posts'),
        fetch('/api/comments')
    ]);

    const [user, posts, comments] = await Promise.all([
        userRes.json(),
        postsRes.json(),
        commentsRes.json()
    ]);

    return { user, posts, comments };
}
```

### Promise.race

```javascript
async function fetchWithTimeout(url, timeout) {
    const fetchPromise = fetch(url);
    const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeout)
    );

    return await Promise.race([fetchPromise, timeoutPromise]);
}
```

## Best Practices

### 1. Always Handle Errors

```javascript
// GOOD: Handle errors
async function safeOperation() {
    try {
        await riskyOperation();
    } catch (error) {
        console.error(error);
    }
}
```

### 2. Use Promise.all for Parallel

```javascript
// GOOD: Parallel
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// BAD: Sequential
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();
```

## Internal Mechanisms

### Async/Await의 Generator 변환

async/await는 내부적으로 Generator + Promise로 변환됩니다:

```javascript
// 원본 async 함수
async function fetchUser(id) {
    const response = await fetch(`/user/${id}`);
    const user = await response.json();
    return user;
}

// 컴파일러 변환 결과 (개념적)
function fetchUser(id) {
    return new Promise((resolve, reject) => {
        const generator = function* () {
            try {
                const response = yield fetch(`/user/${id}`);
                const user = yield response.json();
                return user;
            } catch (error) {
                throw error;
            }
        }();

        function step(nextFn) {
            let result;
            try {
                result = nextFn();
            } catch (error) {
                return reject(error);
            }

            if (result.done) {
                return resolve(result.value);
            }

            // result.value는 Promise
            Promise.resolve(result.value).then(
                value => step(() => generator.next(value)),
                error => step(() => generator.throw(error))
            );
        }

        step(() => generator.next());
    });
}
```

### V8 Engine의 Async 최적화

```
Zero-Cost Async Stack Traces (V8 7.3+):
┌─────────────────────────────────────────────────────────────┐
│ 기존 방식:                                                  │
│ - 각 await에서 스택 트레이스 캡처                           │
│ - 메모리 사용량 높음                                        │
│                                                             │
│ 최적화된 방식:                                              │
│ - Promise에 async function 참조만 저장                      │
│ - 에러 발생 시에만 스택 재구성                              │
│ - 일반 실행 시 오버헤드 없음                                │
└─────────────────────────────────────────────────────────────┘

Async Function 상태 머신:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [시작] ──▶ [await 1] ──▶ [await 2] ──▶ ... ──▶ [완료]     │
│     │           │            │                   │         │
│     │     suspend/resume  suspend/resume         │         │
│     │           │            │                   │         │
│     └───────────┴────────────┴───────────────────┘         │
│                    Promise chain                            │
└─────────────────────────────────────────────────────────────┘
```

### Promise 내부 구조 (V8)

```javascript
// Promise 상태
const PENDING = 0;
const FULFILLED = 1;
const REJECTED = 2;

// Promise 내부 슬롯 (개념적)
// [[PromiseState]]:      pending, fulfilled, rejected
// [[PromiseResult]]:     결과값 또는 에러
// [[PromiseFulfillReactions]]:  then 핸들러 목록
// [[PromiseRejectReactions]]:   catch 핸들러 목록

// then() 호출 시:
// 1. 새 Promise 생성
// 2. reaction 객체 생성 (handler + 새 Promise)
// 3. 상태에 따라:
//    - pending: reaction을 큐에 추가
//    - fulfilled/rejected: microtask로 핸들러 스케줄
```

### await 표현식의 미세 동작

```javascript
async function example() {
    console.log('A');
    await null;  // 심지어 null도 Promise로 래핑됨
    console.log('B');
}

example();
console.log('C');

// 실행 순서:
// 1. example() 호출
// 2. 'A' 출력
// 3. await null → Promise.resolve(null).then(() => resume)
//    - 현재 함수 suspend
//    - microtask queue에 resume 등록
// 4. 동기 코드 계속 → 'C' 출력
// 5. Call Stack 비어짐 → microtask 실행
// 6. resume → 'B' 출력

// 출력: A, C, B
```

## Complete Example

```javascript
class DataFetcher {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
    }

    async fetch(endpoint) {
        try {
            const response = await fetch(`${this.baseUrl}${endpoint}`);

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }

            return await response.json();
        } catch (error) {
            console.error(`Fetch error: ${error}`);
            throw error;
        }
    }

    async fetchAll(endpoints) {
        const promises = endpoints.map(endpoint => this.fetch(endpoint));
        return await Promise.all(promises);
    }
}

// Usage
const fetcher = new DataFetcher('https://api.example.com');

(async () => {
    try {
        const [users, posts] = await fetcher.fetchAll(['/users', '/posts']);
        console.log('Users:', users);
        console.log('Posts:', posts);
    } catch (error) {
        console.error('Error fetching data:', error);
    }
})();
```

## Navigation

- [Back to JavaScript Overview](./README.md)
- Previous: [Event Loop](./01-event-loop.md)
- Next: [Web Workers](./03-web-workers.md)
