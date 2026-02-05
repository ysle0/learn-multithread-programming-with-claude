# JavaScript의 Async/Await

Async/await는 Promise 위에 구축된, 동기 코드처럼 보이는 깔끔한 비동기 연산 문법을 제공합니다.

## 기본 개념

### Async 함수

```javascript
// Async 함수는 항상 Promise를 반환
async function fetchData() {
    return 'data';
}

fetchData().then(data => console.log(data));  // 'data'

// 다음과 동일:
function fetchDataOld() {
    return Promise.resolve('data');
}
```

### Await 표현식

```javascript
async function getData() {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
}

// 다음과 동일:
function getDataOld() {
    return fetch('https://api.example.com/data')
        .then(response => response.json());
}
```

## 에러 처리

### Try-Catch

```javascript
async function fetchWithErrorHandling() {
    try {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error('HTTP 에러');
        }
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('에러:', error);
        return null;
    }
}
```

## 병렬 연산

### Promise.all

```javascript
async function fetchMultiple() {
    // 순차적 (느림) - 총 6초
    const user = await fetch('/api/user');
    const posts = await fetch('/api/posts');
    const comments = await fetch('/api/comments');

    // 병렬 (빠름) - 총 2초
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
        setTimeout(() => reject(new Error('시간 초과')), timeout)
    );

    return await Promise.race([fetchPromise, timeoutPromise]);
}
```

## 모범 사례

### 1. 항상 에러를 처리할 것

```javascript
// 좋은 예: 에러 처리
async function safeOperation() {
    try {
        await riskyOperation();
    } catch (error) {
        console.error(error);
    }
}
```

### 2. 병렬 작업에는 Promise.all 사용

```javascript
// 좋은 예: 병렬
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// 나쁜 예: 순차적
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();
```

## 내부 메커니즘

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

## 완전한 예제

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
            console.error(`Fetch 에러: ${error}`);
            throw error;
        }
    }

    async fetchAll(endpoints) {
        const promises = endpoints.map(endpoint => this.fetch(endpoint));
        return await Promise.all(promises);
    }
}

// 사용법
const fetcher = new DataFetcher('https://api.example.com');

(async () => {
    try {
        const [users, posts] = await fetcher.fetchAll(['/users', '/posts']);
        console.log('사용자:', users);
        console.log('게시물:', posts);
    } catch (error) {
        console.error('데이터 가져오기 에러:', error);
    }
})();
```

## 내비게이션

- [JavaScript 개요로 돌아가기](./README.md)
- 이전: [Event Loop](./01-event-loop.md)
- 다음: [Web Workers](./03-web-workers.md)
