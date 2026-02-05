# Node.js의 Worker Threads

Worker threads는 Node.js에서 멀티스레딩 기능을 제공하며, Web Workers와 유사하지만 일부 Node.js 전용 기능이 있습니다.

## 기본 사용법

```javascript
// main.js
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js', {
    workerData: { value: 42 }
});

worker.on('message', (result) => {
    console.log('결과:', result);
});

worker.on('error', (error) => {
    console.error('에러:', error);
});

worker.on('exit', (code) => {
    console.log(`Worker가 코드 ${code}로 종료됨`);
});

// worker.js
const { parentPort, workerData } = require('worker_threads');

const result = workerData.value * 2;
parentPort.postMessage(result);
```

## 데이터 공유

### MessageChannel

```javascript
const { Worker, MessageChannel } = require('worker_threads');

const { port1, port2 } = new MessageChannel();

const worker = new Worker('./worker.js');
worker.postMessage({ port: port2 }, [port2]);

port1.on('message', (msg) => {
    console.log('수신:', msg);
});

port1.postMessage('메인에서 보냄');
```

## 완전한 예제: Worker Pool

```javascript
const { Worker } = require('worker_threads');

class WorkerPool {
    constructor(workerScript, poolSize = 4) {
        this.workerScript = workerScript;
        this.poolSize = poolSize;
        this.workers = [];
        this.freeWorkers = [];
        this.queue = [];

        this.init();
    }

    init() {
        for (let i = 0; i < this.poolSize; i++) {
            this.addWorker();
        }
    }

    addWorker() {
        const worker = new Worker(this.workerScript);
        worker.on('message', (result) => this.handleResult(worker, result));
        worker.on('error', (error) => console.error(error));
        this.workers.push(worker);
        this.freeWorkers.push(worker);
    }

    handleResult(worker, result) {
        const { resolve, reject } = worker.currentTask;
        delete worker.currentTask;

        this.freeWorkers.push(worker);
        resolve(result);

        this.processQueue();
    }

    exec(data) {
        return new Promise((resolve, reject) => {
            const task = { data, resolve, reject };

            if (this.freeWorkers.length > 0) {
                this.runTask(task);
            } else {
                this.queue.push(task);
            }
        });
    }

    runTask(task) {
        const worker = this.freeWorkers.pop();
        worker.currentTask = task;
        worker.postMessage(task.data);
    }

    processQueue() {
        if (this.queue.length > 0 && this.freeWorkers.length > 0) {
            const task = this.queue.shift();
            this.runTask(task);
        }
    }

    destroy() {
        this.workers.forEach(worker => worker.terminate());
    }
}

// 사용법
const pool = new WorkerPool('./compute-worker.js', 4);

Promise.all([
    pool.exec({ n: 10 }),
    pool.exec({ n: 20 }),
    pool.exec({ n: 30 })
]).then(results => {
    console.log('결과:', results);
    pool.destroy();
});
```

## 내부 메커니즘

### Node.js Worker 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Main Thread                                 │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ V8 Isolate + libuv Event Loop                                 │ │
│  │      │                                                        │ │
│  │      │ MessagePort (내부적으로 libuv pipe)                    │ │
│  └──────┼────────────────────────────────────────────────────────┘ │
└─────────┼───────────────────────────────────────────────────────────┘
          │
          │ Structured Clone / Transfer
          │
┌─────────┼───────────────────────────────────────────────────────────┐
│         ▼                                                           │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ V8 Isolate (별도) + libuv Event Loop (별도)                   │ │
│  │                                                               │ │
│  │ - 같은 프로세스 내 별도 스레드                                 │ │
│  │ - 메모리 격리 (SharedArrayBuffer 제외)                        │ │
│  │ - require() 가능                                              │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                         Worker Thread                               │
└─────────────────────────────────────────────────────────────────────┘
```

### workerData vs postMessage

```javascript
// workerData: 생성 시 한 번만 전달 (복사)
const worker = new Worker('./worker.js', {
    workerData: { config: 'initial' }  // 즉시 사용 가능
});

// postMessage: 언제든 전달 가능
worker.postMessage({ dynamic: 'data' });

// Worker 내부:
const { workerData, parentPort } = require('worker_threads');
console.log(workerData);  // { config: 'initial' } - 즉시 접근 가능

parentPort.on('message', (msg) => {
    console.log(msg);  // { dynamic: 'data' }
});
```

### Worker Thread vs Child Process

```
Worker Thread:
- 같은 프로세스, 다른 스레드
- 메모리 공유 가능 (SharedArrayBuffer)
- 더 빠른 생성/통신
- Node.js 12+ 안정화

Child Process (child_process):
- 별도 프로세스
- 메모리 완전 격리
- IPC 통신 (JSON 직렬화)
- 더 높은 안정성 (한 프로세스 크래시 무관)
```

## 내비게이션

- [JavaScript 개요로 돌아가기](./README.md)
- 이전: [Web Workers](./03-web-workers.md)
- 다음: [SharedArrayBuffer](./05-shared-array-buffer.md)
