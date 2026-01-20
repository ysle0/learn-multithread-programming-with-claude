# Worker Threads in Node.js

Worker threads provide multi-threading capabilities in Node.js, similar to Web Workers but with some Node-specific features.

## Basic Usage

```javascript
// main.js
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js', {
    workerData: { value: 42 }
});

worker.on('message', (result) => {
    console.log('Result:', result);
});

worker.on('error', (error) => {
    console.error('Error:', error);
});

worker.on('exit', (code) => {
    console.log(`Worker exited with code ${code}`);
});

// worker.js
const { parentPort, workerData } = require('worker_threads');

const result = workerData.value * 2;
parentPort.postMessage(result);
```

## Sharing Data

### MessageChannel

```javascript
const { Worker, MessageChannel } = require('worker_threads');

const { port1, port2 } = new MessageChannel();

const worker = new Worker('./worker.js');
worker.postMessage({ port: port2 }, [port2]);

port1.on('message', (msg) => {
    console.log('Received:', msg);
});

port1.postMessage('Hello from main');
```

## Complete Example: Worker Pool

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

// Usage
const pool = new WorkerPool('./compute-worker.js', 4);

Promise.all([
    pool.exec({ n: 10 }),
    pool.exec({ n: 20 }),
    pool.exec({ n: 30 })
]).then(results => {
    console.log('Results:', results);
    pool.destroy();
});
```

## Navigation

- [Back to JavaScript Overview](./README.md)
- Previous: [Web Workers](./03-web-workers.md)
- Next: [SharedArrayBuffer](./05-shared-array-buffer.md)
