# JavaScript의 Web Workers

Web Workers는 백그라운드 스레드에서 스크립트를 실행하여 브라우저에서 진정한 멀티스레딩을 가능하게 합니다.

## 기본 사용법

### Worker 생성

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'start', data: [1, 2, 3] });

worker.onmessage = (event) => {
    console.log('결과:', event.data);
};

worker.onerror = (error) => {
    console.error('Worker 에러:', error);
};

// worker.js
self.onmessage = (event) => {
    const { type, data } = event.data;

    if (type === 'start') {
        const result = processData(data);
        self.postMessage(result);
    }
};

function processData(data) {
    return data.map(x => x * 2);
}
```

## Transferable Objects

### 제로 카피 전송

```javascript
// main.js
const buffer = new ArrayBuffer(1024);
const view = new Uint8Array(buffer);

// 소유권 이전 (제로 카피)
worker.postMessage(buffer, [buffer]);
// buffer는 이제 무효화됨, 사용 불가

// worker.js
self.onmessage = (event) => {
    const buffer = event.data;
    const view = new Uint8Array(buffer);
    // buffer 처리
};
```

## Worker 유형

### Dedicated Worker

```javascript
// 단일 페이지가 worker 사용
const worker = new Worker('worker.js');
```

### Shared Worker

```javascript
// 여러 페이지가 공유 가능
const worker = new SharedWorker('shared-worker.js');

worker.port.onmessage = (event) => {
    console.log(event.data);
};

worker.port.postMessage('hello');
```

## 완전한 예제: 이미지 처리

```javascript
// main.js
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

const worker = new Worker('image-worker.js');

worker.postMessage({
    imageData: imageData,
    effect: 'grayscale'
});

worker.onmessage = (event) => {
    ctx.putImageData(event.data, 0, 0);
    worker.terminate();
};

// image-worker.js
self.onmessage = (event) => {
    const { imageData, effect } = event.data;
    const data = imageData.data;

    if (effect === 'grayscale') {
        for (let i = 0; i < data.length; i += 4) {
            const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
            data[i] = avg;      // 빨강
            data[i + 1] = avg;  // 초록
            data[i + 2] = avg;  // 파랑
        }
    }

    self.postMessage(imageData);
};
```

## 내부 메커니즘

### Worker 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Main Thread                                │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    V8 Isolate (Main)                          │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │ │
│  │  │    Heap     │  │    Stack    │  │   Event Loop        │   │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │ │
│  │                          │                                    │ │
│  │              postMessage │ (structured clone)                 │ │
│  └──────────────────────────┼────────────────────────────────────┘ │
└─────────────────────────────┼───────────────────────────────────────┘
                              │ Message Queue (IPC)
┌─────────────────────────────┼───────────────────────────────────────┐
│                          Worker Thread                              │
│  ┌──────────────────────────┼────────────────────────────────────┐ │
│  │                    V8 Isolate (Worker) - 독립된 힙/스택       │ │
│  │                                                               │ │
│  │  - DOM 접근 불가                                              │ │
│  │  - window 객체 없음 (self 사용)                               │ │
│  │  - 별도의 Event Loop                                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Transferable Objects (제로 카피)

```javascript
// 소유권 이전 - 복사 없이 전달
const buffer = new ArrayBuffer(100 * 1024 * 1024);  // 100MB

// 두 번째 인수: transfer list
worker.postMessage(buffer, [buffer]);

// 이제 main thread에서 buffer 접근 불가:
console.log(buffer.byteLength);  // 0 (neutered)

// 전송 가능한 객체:
// - ArrayBuffer
// - MessagePort
// - ImageBitmap
// - OffscreenCanvas
```

### Worker 생성 비용

```
Worker 생성 과정:
1. 새 OS 스레드 생성 (~100-500μs)
2. 새 V8 Isolate 초기화 (~5-10ms)
3. 스크립트 다운로드 (네트워크 지연)
4. 스크립트 파싱 및 컴파일

총 비용: 수십 ms ~ 수백 ms
→ 자주 생성/소멸하지 말고 Worker Pool 사용 권장
```

## 내비게이션

- [JavaScript 개요로 돌아가기](./README.md)
- 이전: [Async/Await](./02-async-await.md)
- 다음: [Worker Threads](./04-worker-threads.md)
