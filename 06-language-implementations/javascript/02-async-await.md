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
