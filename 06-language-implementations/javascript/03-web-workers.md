# Web Workers in JavaScript

Web Workers enable true multi-threading in browsers by running scripts in background threads.

## Basic Usage

### Creating a Worker

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'start', data: [1, 2, 3] });

worker.onmessage = (event) => {
    console.log('Result:', event.data);
};

worker.onerror = (error) => {
    console.error('Worker error:', error);
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

### Zero-Copy Transfer

```javascript
// main.js
const buffer = new ArrayBuffer(1024);
const view = new Uint8Array(buffer);

// Transfer ownership (zero-copy)
worker.postMessage(buffer, [buffer]);
// buffer is now neutered, can't use it

// worker.js
self.onmessage = (event) => {
    const buffer = event.data;
    const view = new Uint8Array(buffer);
    // Process buffer
};
```

## Worker Types

### Dedicated Worker

```javascript
// Single page uses worker
const worker = new Worker('worker.js');
```

### Shared Worker

```javascript
// Multiple pages can share
const worker = new SharedWorker('shared-worker.js');

worker.port.onmessage = (event) => {
    console.log(event.data);
};

worker.port.postMessage('hello');
```

## Complete Example: Image Processing

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
            data[i] = avg;      // Red
            data[i + 1] = avg;  // Green
            data[i + 2] = avg;  // Blue
        }
    }

    self.postMessage(imageData);
};
```

## Navigation

- [Back to JavaScript Overview](./README.md)
- Previous: [Async/Await](./02-async-await.md)
- Next: [Worker Threads](./04-worker-threads.md)
